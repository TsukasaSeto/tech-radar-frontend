# XSS 対策のベストプラクティス

React は標準で文字列を自動エスケープするが、それを迂回する API（`dangerouslySetInnerHTML`・`href` 等）や境界（URL パラメータ・JSON 埋め込み）に攻撃面がある。

## ルール

### 1. `dangerouslySetInnerHTML` は最終手段とし、DOMPurify でサニタイズする

ユーザー由来の HTML を表示する必要があるなら、必ず DOMPurify 等のサニタイザーで通してから渡す。
**信頼できないソースを生で `dangerouslySetInnerHTML` に渡してはならない**。

**根拠**:
- React の JSX 内 `{expression}` は標準でテキストとしてエスケープされ XSS が起きない
- `dangerouslySetInnerHTML` はエスケープを完全にバイパスするため、攻撃者の `<script>` がそのまま実行される
- DOMPurify は 250+ の攻撃ベクトルを既知パターンとして弾く、業界標準のサニタイザー
- 自前の正規表現でサニタイズすると必ず迂回される（`<scr<script>ipt>` 等）

**コード例**:
```tsx
// Bad: 信頼できない HTML をそのまま埋め込む（XSS）
function ArticleBody({ html }: { html: string }) {
  return <div dangerouslySetInnerHTML={{ __html: html }} />;
}

// Good: DOMPurify でサニタイズ
import DOMPurify from 'isomorphic-dompurify';

function ArticleBody({ html }: { html: string }) {
  const clean = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['p', 'a', 'strong', 'em', 'ul', 'ol', 'li', 'h1', 'h2', 'h3', 'blockquote', 'code', 'pre'],
    ALLOWED_ATTR: ['href', 'title'],
    ALLOW_DATA_ATTR: false,
  });
  return <div dangerouslySetInnerHTML={{ __html: clean }} />;
}

// Better: そもそも HTML ではなく構造化データを使う
import ReactMarkdown from 'react-markdown';

function ArticleBody({ markdown }: { markdown: string }) {
  return <ReactMarkdown>{markdown}</ReactMarkdown>;
}
```

**サニタイズはサーバー側でも実施**:
```ts
// Server Action / API でも保存前にサニタイズ（多層防御）
'use server';
import DOMPurify from 'isomorphic-dompurify';

export async function savePost(formData: FormData) {
  const rawHtml = formData.get('content') as string;
  const cleanHtml = DOMPurify.sanitize(rawHtml, { /* config */ });
  await db.post.create({ data: { content: cleanHtml } });
}
```

**判断軸**:
- HTML を表示する必要があるか？ → 多くの場合 Markdown / structured content で代替可能
- ユーザー由来か？ → 自分のチームが書いた CMS コンテンツでも、CMS 経由でユーザー入力が混入する可能性があれば信頼しない
- リッチエディタの出力か？ → エディタ自体（TipTap / Slate）がサニタイズしていても、表示時に再度 DOMPurify

**出典**:
- [OWASP: XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html) (OWASP)
- [DOMPurify](https://github.com/cure53/DOMPurify) (cure53)
- [React Docs: dangerouslySetInnerHTML](https://react.dev/reference/react-dom/components/common#dangerously-setting-the-inner-html) (React 公式)

**バージョン**: DOMPurify 3+
**確信度**: 高
**最終更新**: 2026-05-16

---

### 2. ユーザー入力をそのまま `href` / `src` に渡さない（`javascript:` スキーム XSS）

`<a href={userUrl}>` のようにユーザー入力を URL に渡すと、`javascript:alert(1)` のようなスキームで XSS が発生する。
URL のスキームを必ず allow list で検証する。

**根拠**:
- React は `href` を文字列としてエスケープするが、**スキーム自体は検証しない**
- `javascript:` `data:` `vbscript:` 等はクリックでスクリプト実行に直結する
- React 19 以降は `javascript:` スキームに警告が出るが、エラーではなく動く
- URL parser を使うのが最も確実。スキームと host を分離して検証する

**コード例**:
```tsx
// Bad: ユーザー入力を href に直接渡す
function ProfileLink({ url }: { url: string }) {
  return <a href={url}>プロフィール</a>;  // <a href="javascript:steal()">
}

// Good: スキーム allow list で検証
const ALLOWED_SCHEMES = ['http:', 'https:', 'mailto:', 'tel:'];

function isSafeUrl(url: string): boolean {
  try {
    const u = new URL(url, 'https://example.com');  // 相対URL対応のため base 指定
    return ALLOWED_SCHEMES.includes(u.protocol);
  } catch {
    return false;
  }
}

function ProfileLink({ url }: { url: string }) {
  if (!isSafeUrl(url)) return <span>無効なURL</span>;
  return <a href={url} rel="noopener noreferrer">プロフィール</a>;
}

// 外部リンクには noopener noreferrer を付ける（target="_blank" 時の opener 攻撃対策）
<a href={externalUrl} target="_blank" rel="noopener noreferrer">
```

**`src` 属性も同様**:
```tsx
// Bad: ユーザー入力画像 URL
<img src={userImageUrl} />  // <img src="javascript:..."> は古い IE でしか動かないが、data: URL で SVG XSS が可能

// Good: 信頼できる CDN ドメインのみ許可
function isSafeImageUrl(url: string): boolean {
  try {
    const u = new URL(url);
    if (!['https:'].includes(u.protocol)) return false;
    const ALLOWED_HOSTS = ['cdn.example.com', 'images.example.com'];
    return ALLOWED_HOSTS.includes(u.host);
  } catch {
    return false;
  }
}
```

**Next.js Image の場合**:
```ts
// next.config.ts で remotePatterns を明示
export default {
  images: {
    remotePatterns: [
      { protocol: 'https', hostname: 'cdn.example.com' },
      { protocol: 'https', hostname: 'images.example.com' },
    ],
  },
};
// 上記以外のドメインは <Image> が拒否する（追加の URL 検証層になる）
```

**SVG ファイルの注意**:
- ユーザーがアップロードした SVG を `<img>` で表示する → SVG 内の `<script>` は実行されない（safe）
- 同 SVG を `dangerouslySetInnerHTML` で **インライン埋め込み** すると `<script>` が実行される → XSS
- SVG は常に `<img src>` または `<object>` で外部読み込みとして扱う

**出典**:
- [OWASP: DOM-based XSS](https://owasp.org/www-community/attacks/DOM_Based_XSS) (OWASP)
- [React Docs: javascript: URL warning](https://react.dev/reference/react-dom/components/common#applying-css-styles) (React 公式)
- [MDN: URL constructor](https://developer.mozilla.org/en-US/docs/Web/API/URL/URL) (MDN Web Docs)

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-05-16

---

### 3. JSON は `<script>` に直接埋め込まず、エスケープしてから埋め込む

Server Component から Client にデータを渡す目的で `<script type="application/json">` 等にデータを埋め込む際、
`</script>` 文字列を含む JSON はタグを早期終了させて XSS を発生させる。`<` `>` `&` `'` をエスケープする。

**根拠**:
- `JSON.stringify({ name: '</script><script>alert(1)</script>' })` の出力には `</script>` がそのまま含まれる
- ブラウザは `</script>` を見つけた時点で `<script>` を終了するため、後続のスクリプトが実行される
- `<` `>` `&` を Unicode エスケープ（`<` `>` `&`）すると JSON としては有効なまま、HTML としては安全になる
- Next.js の `Script` コンポーネントは内部で対策済みだが、自前で書く場合は注意

**コード例**:
```tsx
// Bad: そのまま埋め込む
function StateScript({ state }: { state: unknown }) {
  return (
    <script
      id="__APP_STATE__"
      type="application/json"
      dangerouslySetInnerHTML={{ __html: JSON.stringify(state) }}
    />
  );
}
// state に '</script><script>alert(1)</script>' が含まれると XSS

// Good: HTML として危険な文字をエスケープ
function escapeJsonForHtml(json: string): string {
  return json
    .replace(/</g, '\\u003c')
    .replace(/>/g, '\\u003e')
    .replace(/&/g, '\\u0026')
    .replace(/'/g, '\\u0027')
    .replace(/ /g, '\\u2028')  // 行区切り
    .replace(/ /g, '\\u2029'); // 段落区切り
}

function StateScript({ state }: { state: unknown }) {
  const safeJson = escapeJsonForHtml(JSON.stringify(state));
  return (
    <script
      id="__APP_STATE__"
      type="application/json"
      dangerouslySetInnerHTML={{ __html: safeJson }}
    />
  );
}

// クライアント側での読み取り
const script = document.getElementById('__APP_STATE__');
const state = JSON.parse(script?.textContent ?? '{}');
```

**ライブラリでの対策**:
- `serialize-javascript`: 上記エスケープに加え、Date / RegExp / 関数のシリアライズを安全に行う
- Next.js `<Script>`: 自動エスケープ済み（明示的に指定不要）

**判断軸（そもそも埋め込みが必要か）**:
- Next.js では Server Component から Client Component へは props 経由でデータが渡せる
- 「初期データを HTML に埋め込み、クライアントで読み取る」パターンは hydration 用の限定ケース
- App Router では React Server Components が自動的にこのシリアライズを安全に行う

**出典**:
- [OWASP: XSS Prevention Cheat Sheet - HTML JavaScript Encoding](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html#output-encoding-rules-summary) (OWASP)
- [serialize-javascript](https://github.com/yahoo/serialize-javascript) (Yahoo)
- [JSON.stringify pitfalls](https://v8.dev/features/subsume-json) (V8)

**バージョン**: 全モダンブラウザ
**確信度**: 高
**最終更新**: 2026-05-16

---

### 4. URL パラメータの reflection は必ずエスケープする

`useSearchParams()` で取得した値をそのまま画面に表示するのは React の自動エスケープで安全だが、
属性値・スタイル・スクリプト・URL 等に展開する際は別途エスケープが必要。

**根拠**:
- JSX 内 `{searchParam}` のテキスト埋め込みは安全（React がエスケープ）
- ただし `style={{ background: searchParam }}` や `<a href={searchParam}>` は React が検証しない
- 攻撃者は `?q=" onerror="alert(1)` のようなクエリ文字列で属性ブレイクアウトを試みる
- Reflected XSS は OWASP Top 10 で最も多い XSS 種別

**コード例**:
```tsx
'use client';
import { useSearchParams } from 'next/navigation';

function SearchPage() {
  const params = useSearchParams();
  const q = params.get('q') ?? '';

  return (
    <>
      {/* Safe: テキストとして表示（React 自動エスケープ） */}
      <p>「{q}」の検索結果</p>

      {/* Bad: スタイル属性に展開（CSS injection） */}
      <div style={{ background: `url(${q})` }}>...</div>

      {/* Bad: 属性に展開せざるを得ない場合のあり方 */}
      <a href={q}>戻る</a>

      {/* Good: 検証してから使う */}
      {isSafeUrl(q) && <a href={q}>戻る</a>}

      {/* Good: 必要なら whitelist 形式 */}
      <a href={['/', '/about', '/contact'].includes(q) ? q : '/'}>戻る</a>
    </>
  );
}
```

**Server Component での扱い**:
```tsx
// page.tsx
type Props = { searchParams: Promise<{ q?: string }> };

export default async function SearchPage({ searchParams }: Props) {
  const { q = '' } = await searchParams;

  // Safe: テキスト表示
  return <h1>{q}</h1>;

  // Bad: meta タグに展開（HTML injection）
  // <head> に動的に title を入れたい場合
}

// Good: generateMetadata は内部で適切にエスケープされる
export async function generateMetadata({ searchParams }: Props): Promise<Metadata> {
  const { q = '' } = await searchParams;
  return {
    title: `${q} の検索結果`,  // Next.js が自動エスケープ
  };
}
```

**チェックポイント**:
- URL パラメータを `<a href>` / `<img src>` に渡す → スキーム検証（Rule 2）
- URL パラメータを `style` / `className` / `data-*` に渡す → 値の文字種を validate
- URL パラメータを SQL / API クエリに渡す → そちらでも別途エスケープ（サーバー責務）
- URL パラメータをエラーメッセージ等で reflect する → React の `{}` 経由のみ

**出典**:
- [OWASP: Reflected XSS](https://owasp.org/www-community/attacks/xss/#reflected-xss-attacks) (OWASP)
- [Next.js Docs: useSearchParams](https://nextjs.org/docs/app/api-reference/functions/use-search-params) (Next.js 公式)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-16

---

### 5. Content-Type を明示し、MIME sniffing 経由の XSS を塞ぐ

レスポンスには `Content-Type` を明示し、`X-Content-Type-Options: nosniff` ヘッダーで MIME sniffing を無効化する。
ブラウザが Content-Type を推測すると、画像として保存した HTML が `<img>` 経由で実行されるパターンが発生する。

**根拠**:
- IE / 旧 Chrome は Content-Type が `text/plain` でも内容が HTML っぽければ HTML として解釈する（MIME sniffing）
- これにより、ユーザーアップロードファイルから XSS が発生する経路があった（画像アップロード機能）
- `X-Content-Type-Options: nosniff` で MIME sniffing を無効化し、Content-Type を尊重させる
- 全モダンブラウザがサポート、デフォルト推奨ヘッダー

**コード例**:
```ts
// Next.js: next.config.ts のヘッダー設定
export default {
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          { key: 'X-Content-Type-Options', value: 'nosniff' },
          { key: 'X-Frame-Options', value: 'DENY' },
          { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
          { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
          // HSTS は HTTPS 全面適用後に
          { key: 'Strict-Transport-Security', value: 'max-age=63072000; includeSubDomains; preload' },
        ],
      },
    ];
  },
};
```

**API レスポンスでも Content-Type を明示**:
```ts
// app/api/data/route.ts
export async function GET() {
  return new Response(JSON.stringify({ ok: true }), {
    status: 200,
    headers: {
      'Content-Type': 'application/json; charset=utf-8',
      'X-Content-Type-Options': 'nosniff',
    },
  });
}
```

**ユーザーアップロードファイルの配信**:
- ユーザーアップロード画像は **別ドメイン** から配信する（`uploads.example.com` 等）
- 配信時は `Content-Disposition: attachment` を付けて inline 表示を防ぐ
- ファイル名は uuid 化し、拡張子だけで Content-Type を決めない（中身を検証）

**関連セキュリティヘッダー（最低限のセット）**:
| ヘッダー | 値 | 効果 |
|---|---|---|
| `Content-Security-Policy` | strict CSP | XSS 多層防御（`csp.md` 参照） |
| `X-Content-Type-Options` | `nosniff` | MIME sniffing 無効化 |
| `X-Frame-Options` | `DENY` または `SAMEORIGIN` | clickjacking 防御 |
| `Strict-Transport-Security` | `max-age=63072000; includeSubDomains` | HTTPS 強制 |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | リファラ漏洩防止 |
| `Permissions-Policy` | feature を明示的に拒否 | カメラ・位置情報等の権限制御 |

**出典**:
- [MDN: X-Content-Type-Options](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Content-Type-Options) (MDN Web Docs)
- [OWASP: Secure Headers Project](https://owasp.org/www-project-secure-headers/) (OWASP)
- [Next.js Docs: Security Headers](https://nextjs.org/docs/app/building-your-application/configuring/content-security-policy#applying-the-csp-header) (Next.js 公式)

**バージョン**: 全モダンブラウザ
**確信度**: 高
**最終更新**: 2026-05-16
