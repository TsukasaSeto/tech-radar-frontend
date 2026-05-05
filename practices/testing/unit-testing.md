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

**出典**:
- [Testing Implementation Details](https://kentcdodds.com/blog/testing-implementation-details) (Kent C. Dodds)
- [Write tests. Not too many. Mostly integration.](https://kentcdodds.com/blog/write-tests) (Kent C. Dodds)

**バージョン**: Vitest 1+, Jest 27+
**確信度**: 高
**最終更新**: 2026-05-05

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
