# ユニットテストのベストプラクティス

## ルール

### 1. テストは振る舞い（behavior）をテストし、実装詳細をテストしない

「何をするか」ではなく「何ができるか」をテストする。
内部のメソッド名・変数名・状態の持ち方に依存したテストを書かない。

**根拠**:
- 実装詳細に依存したテストはリファクタリング時に壊れやすく、メンテコストが高い
- 振る舞いをテストすることで、実装を変えてもテストが生き続ける
- テストが壊れたとき「ロジックが壊れた」のか「リファクタリングしただけ」なのかが明確になる

**コード例**:
```ts
// Bad: 実装詳細のテスト
it('should call _formatPrice internal method', () => {
  const cart = new ShoppingCart();
  expect(cart._formatPrice).toHaveBeenCalled();  // プライベートな実装詳細
});

// Good: 振る舞いのテスト
it('should display formatted price with currency symbol', () => {
  const cart = new ShoppingCart([{ price: 1000, quantity: 2 }]);
  expect(cart.getTotalDisplay()).toBe('¥2,000');  // ユーザーが見る結果
});

// Bad: React の内部状態をテスト
it('should set isLoading state to true', () => {
  const { result } = renderHook(() => useSubmit());
  expect(result.current.isLoading).toBe(true);  // 内部の state 名に依存
});

// Good: ユーザーが体験する振る舞いをテスト
it('should show loading indicator while submitting', async () => {
  render(<SubmitForm />);
  fireEvent.click(screen.getByRole('button', { name: '送信' }));
  expect(screen.getByRole('status')).toBeInTheDocument();  // UIの振る舞い
});
```

**実装詳細テストの 8 つの兆候（コードレビュー時のチェック）**:

以下のいずれかに該当するテストは「実装詳細をテストしている」可能性が高い。リファクタリング耐性が低いため、書き換えを検討する。

1. **`_` プレフィックスの private メソッドを直接呼ぶ** — 例: `expect(cart._calculate).toHaveBeenCalled()` → ユーザーが触れない関数。public API 経由でテスト
2. **`useState` の戻り値の `setState` を直接アサート** — 例: `expect(setState).toHaveBeenCalledWith('loading')` → 内部 state を観測している。レンダリング結果をテスト
3. **コンポーネントの内部変数名・hook 名に依存** — 例: `result.current.isLoading` を直接見る → 名前変更でテスト崩壊。`screen.getByRole('status')` 等の UI 経由
4. **Mock の呼び出し回数を厳密に検証** — 例: `expect(fetcher).toHaveBeenCalledTimes(1)` → リトライ実装の変更で壊れる。最終的な結果（"ユーザーにデータが表示される"）を検証
5. **CSS クラス名・テストID・data-\* 属性で要素を取得** — 例: `getByTestId('submit-btn')` → クラスリネームで壊れる。`getByRole('button', { name: '送信' })` 等のセマンティック検索
6. **スナップショットテストのみで振る舞いを保証** — DOM ツリー全体のスナップショットは「何が壊れたか」を伝えない（`test-strategy.md` Rule 4 参照）
7. **コンポーネントの内部 import をモック** — 例: 子コンポーネントを `vi.mock` で全部スタブ化 → 統合の壊れを検出できない。MSW で外部 API のみモック
8. **アサーションが実装と 1:1 対応** — 「コードの言い換え」になっている。例: `if (x > 0) { add(); }` というロジックに対して `if x > 0 のとき add が呼ばれる` のテスト → ロジックリファクタで即座に壊れる

**書き換えガイド**:

| 兆候 | 書き換え方針 |
|---|---|
| 内部メソッド呼び出し | public API / UI を呼んで結果を検証 |
| setState アサート | `screen.findBy*` で表示される結果を検証 |
| Hook 内部 state | `<TestComponent>` で使い、UI 出力を検証 |
| Mock 回数 | 「最終結果」（DB 状態 / UI 表示）を検証 |
| testid | `getByRole` + `name` で意味的に検索 |
| スナップショット | 重要な要素だけを個別 assert |
| 子コンポーネント mock | MSW で API のみモック、子は本物を使う |
| 1:1 対応 | テスト名に「何のために」を書き、結果のみ assert |

**「振る舞い」の定義**:

ユーザー（人間 or 上位コード）から観測可能な入出力。具体的には:
- レンダリングされる DOM（テキスト・ロール・aria 属性）
- 発火するイベント・URL 遷移
- 外部 API へのリクエスト内容（method / URL / body）
- ローカルストレージ・Cookie・DB の最終状態

**出典**:
- [Testing Implementation Details](https://kentcdodds.com/blog/testing-implementation-details) (Kent C. Dodds)
- [Write tests. Not too many. Mostly integration.](https://kentcdodds.com/blog/write-tests) (Kent C. Dodds)
- [Testing Library: Guiding Principles](https://testing-library.com/docs/guiding-principles) (Testing Library 公式)
- [Common mistakes with React Testing Library](https://kentcdodds.com/blog/common-mistakes-with-react-testing-library) (Kent C. Dodds)

**バージョン**: Vitest 1+, Jest 27+, Testing Library 14+
**確信度**: 高
**最終更新**: 2026-05-05 / 補強 2026-05-16

---

### 2. AAA（Arrange・Act・Assert）パターンでテストを構造化する

テストを「準備（Arrange）→ 実行（Act）→ 検証（Assert）」の3フェーズで明確に書く。

**根拠**:
- テストの読みやすさが向上し、何をテストしているか一目で分かる
- フェーズが混在した「スパゲッティテスト」を防げる
- CI失敗時にどのフェーズで落ちたか即座に把握できる

**コード例**:
```ts
import { calculateDiscount } from '@/lib/pricing';

describe('calculateDiscount', () => {
  it('should apply 10% discount for premium members', () => {
    // Arrange: テストデータと前提条件を準備
    const price = 10000;
    const user = { tier: 'premium' as const };

    // Act: テスト対象を実行
    const discountedPrice = calculateDiscount(price, user);

    // Assert: 結果を検証
    expect(discountedPrice).toBe(9000);
  });

  it('should not apply discount for regular members', () => {
    // Arrange
    const price = 10000;
    const user = { tier: 'regular' as const };

    // Act
    const discountedPrice = calculateDiscount(price, user);

    // Assert
    expect(discountedPrice).toBe(10000);
  });

  it('should throw for negative price', () => {
    // Arrange
    const price = -100;
    const user = { tier: 'regular' as const };

    // Act & Assert (例外の場合はまとめることが多い)
    expect(() => calculateDiscount(price, user)).toThrow('Price must be non-negative');
  });
});
```

**出典**:
- [Arrange-Act-Assert](https://docs.microsoft.com/en-us/visualstudio/test/unit-test-basics) (Microsoft Docs)

**バージョン**: Vitest 1+, Jest 27+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. `describe` でテストをグループ化し、`it` の説明を「should ～」形式で書く

テストの説明は仕様書として読めるように書く。
`describe` でテスト対象をグループ化し、`it` で具体的なシナリオを記述する。

**根拠**:
- テストが実行可能な仕様書（Living Documentation）として機能する
- テスト失敗時のエラーメッセージが「何が期待されているか」を明確に伝える
- `describe` のネストでエッジケースを整理できる

**コード例**:
```ts
// vitest / jest 共通
describe('formatCurrency', () => {
  describe('日本円のフォーマット', () => {
    it('should format positive amount with yen symbol', () => {
      expect(formatCurrency(1000, 'JPY')).toBe('¥1,000');
    });

    it('should format zero as ¥0', () => {
      expect(formatCurrency(0, 'JPY')).toBe('¥0');
    });

    it('should format large amount with comma separators', () => {
      expect(formatCurrency(1000000, 'JPY')).toBe('¥1,000,000');
    });
  });

  describe('無効な入力', () => {
    it('should throw for negative amount', () => {
      expect(() => formatCurrency(-1, 'JPY')).toThrow(RangeError);
    });

    it('should throw for unsupported currency code', () => {
      expect(() => formatCurrency(100, 'XYZ')).toThrow('Unsupported currency');
    });
  });
});
```

**出典**:
- [Vitest Docs: Test Organization](https://vitest.dev/guide/) (Vitest公式)

**バージョン**: Vitest 1+, Jest 27+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. `vi.mock` と `vi.spyOn` を目的に応じて使い分ける

モジュール全体を差し替える場合は `vi.mock`、既存オブジェクトの一部メソッドだけを監視・置換する場合は `vi.spyOn` を使う。
用途を混同するとテストの意図が不明確になり、意図しない副作用が生じる。

**根拠**:
- `vi.mock` はモジュール解決時点でホイストされ、インポート後の全呼び出しを置換するため副作用の完全な排除に向く
- `vi.spyOn` は元の実装を保持しつつ呼び出しを記録できるため、実装の一部だけを差し替えたい場合に適する
- `vi.spyOn` は `afterEach(() => vi.restoreAllMocks())` と組み合わせることで元の実装への復元が保証される

**コード例**:
```ts
import { vi, describe, it, expect, afterEach } from 'vitest';
import * as api from '@/lib/api';
import { fetchUser } from '@/lib/api';

// vi.mock: モジュール全体を差し替え（副作用を完全に除去したい場合）
vi.mock('@/lib/api', () => ({
  fetchUser: vi.fn().mockResolvedValue({ id: '1', name: 'Alice' }),
  updateUser: vi.fn().mockResolvedValue({ success: true }),
}));

// vi.spyOn: 特定メソッドのみを監視・置換（元実装を保持したい場合）
describe('UserService', () => {
  afterEach(() => {
    vi.restoreAllMocks(); // spyOn で変更したメソッドを元に戻す
  });

  it('should call fetchUser with correct id', async () => {
    // Good: spyOn で特定メソッドだけ監視
    const spy = vi.spyOn(api, 'fetchUser').mockResolvedValue({ id: '42', name: 'Bob' });

    const result = await api.fetchUser('42');

    expect(spy).toHaveBeenCalledWith('42');
    expect(result.name).toBe('Bob');
  });

  it('should NOT use vi.mock when only observing a single method', async () => {
    // Bad: モジュール全体をモックすると他のエクスポートも巻き込む
    // vi.mock('@/lib/api') <- 不要なモック範囲の拡大
  });
});
```

**出典**:
- [Vitest Docs: Mocking](https://vitest.dev/guide/mocking) (Vitest公式)
- [Vitest API: vi.mock / vi.spyOn](https://vitest.dev/api/vi#vi-mock) (Vitest公式)

**バージョン**: Vitest 2+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 5. `vi.useFakeTimers()` でタイマー依存のロジックをコントロールする

`setTimeout`・`setInterval`・`Date` などの時間依存ロジックは `vi.useFakeTimers()` で制御し、
実際の時間待機なしにテストを高速・安定化させる。

**根拠**:
- 実時間を待つテストは不安定（フレーキー）になりやすく CI で失敗しやすい
- `vi.advanceTimersByTime()` で時間を任意に進められるため、debounce・throttle・ポーリングのテストが容易になる
- `vi.setSystemTime()` で現在時刻を固定することで、日付依存のロジックを決定論的にテストできる

**コード例**:
```ts
import { vi, describe, it, expect, beforeEach, afterEach } from 'vitest';
import { debounce } from '@/lib/utils';
import { formatRelativeTime } from '@/lib/date';

describe('debounce', () => {
  beforeEach(() => {
    vi.useFakeTimers(); // タイマーを偽物に切り替え
  });

  afterEach(() => {
    vi.useRealTimers(); // テスト後に必ず元に戻す
  });

  it('should not call function before delay', () => {
    const fn = vi.fn();
    const debounced = debounce(fn, 500);

    debounced();
    vi.advanceTimersByTime(400); // 400ms 進める（まだ呼ばれない）
    expect(fn).not.toHaveBeenCalled();

    vi.advanceTimersByTime(100); // さらに 100ms（合計 500ms）
    expect(fn).toHaveBeenCalledTimes(1);
  });

  it('should reset timer on repeated calls', () => {
    const fn = vi.fn();
    const debounced = debounce(fn, 500);

    debounced();
    vi.advanceTimersByTime(300);
    debounced(); // タイマーリセット
    vi.advanceTimersByTime(300); // 合計 600ms だが最後の呼び出しから 300ms しか経っていない
    expect(fn).not.toHaveBeenCalled();

    vi.advanceTimersByTime(200); // 最後の呼び出しから 500ms 経過
    expect(fn).toHaveBeenCalledTimes(1);
  });
});

describe('formatRelativeTime', () => {
  it('should return "1時間前" for 1 hour ago', () => {
    // 現在時刻を固定
    vi.setSystemTime(new Date('2026-05-06T12:00:00Z'));
    const oneHourAgo = new Date('2026-05-06T11:00:00Z');
    expect(formatRelativeTime(oneHourAgo)).toBe('1時間前');
  });
});
```

**出典**:
- [Vitest API: vi.useFakeTimers](https://vitest.dev/api/vi#vi-usefaketimers) (Vitest公式)
- [Vitest API: vi.advanceTimersByTime](https://vitest.dev/api/vi#vi-advancetimersbyTime) (Vitest公式)

**バージョン**: Vitest 2+
**確信度**: 高
**最終更新**: 2026-05-06
