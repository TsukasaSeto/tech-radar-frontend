# フォント読み込みのベストプラクティス

Web フォントは LCP・CLS の主要な悪化要因の 1 つ。フォーマット選定・配信戦略・フォールバック整形の 4 視点で最適化する。

## ルール

### 1. Web フォントは `font-display: swap` または `optional` で FOIT を回避する

`@font-face` には必ず `font-display` を設定する。
ブロッキング動作（`block`）はデフォルトであり、フォント読み込み中にテキストが表示されない（FOIT: Flash of Invisible Text）状態を引き起こす。

**根拠**:
- `block` はおよそ 3 秒間テキストを非表示にする。LCP が大きく悪化する
- `swap` はフォールバックフォントで即座にテキストを表示し、Web フォント読み込み後に差し替える（FOUT: Flash of Unstyled Text を許容）
- `optional` は 100ms のタイムアウトで諦め、フォールバックのまま継続する。ネットワークが遅いユーザーで CLS が発生しない
- ボディテキストには `swap`、装飾的フォントには `optional` が無難

**コード例**:
```css
/* Bad: font-display なし → デフォルトの block 動作で FOIT */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.woff2') format('woff2');
}

/* Good: 本文用フォントは swap */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.woff2') format('woff2');
  font-display: swap;
  font-weight: 400;
  font-style: normal;
}

/* Good: 装飾フォント（見出し等）は optional */
@font-face {
  font-family: 'DisplayFont';
  src: url('/fonts/display.woff2') format('woff2');
  font-display: optional;
}
```

**Next.js Font Optimization（`next/font`）の場合**:
```tsx
// next/font は自動的にセルフホスト + font-display: swap を適用
import { Inter, Noto_Sans_JP } from 'next/font/google';

const inter = Inter({
  subsets: ['latin'],
  display: 'swap',         // デフォルトだが明示
  preload: true,
  fallback: ['system-ui', 'arial'],
});

const notoSansJP = Noto_Sans_JP({
  subsets: ['latin'],      // 日本語は subset 指定が複雑なため preload を切る選択も
  display: 'swap',
  preload: false,          // 日本語フォントは巨大なので preload しない選択肢
});
```

**font-display の各値**:
| 値 | ブロック期間 | スワップ期間 | 用途 |
|----|--------|--------|------|
| `block` | 3s | 不定 | アイコンフォント等、置換すると壊れる用途のみ |
| `swap` | 0 | 不定 | 本文・見出し（推奨） |
| `fallback` | 100ms | 3s | 中間。フォールバック表示を尊重 |
| `optional` | 100ms | 0 | 装飾的フォント。CLS ゼロが最優先 |

**出典**:
- [MDN: font-display](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face/font-display) (MDN Web Docs)
- [web.dev: Avoid invisible text during font loading](https://web.dev/articles/avoid-invisible-text) (web.dev)
- [web.dev: Controlling font performance with font-display](https://developer.chrome.com/blog/font-display) (Chrome for Developers)

**バージョン**: CSS Fonts Module Level 4 / すべてのモダンブラウザ
**確信度**: 高
**最終更新**: 2026-05-16

---

### 2. 重要フォントは `<link rel="preload" as="font" crossorigin>` でプリロードする

ファーストビュー内で使われる Web フォントは preload して FOUT 時間を短縮する。
ただし `crossorigin` 属性が必須で、誤った指定では二重ダウンロードが発生する。

**根拠**:
- ブラウザは CSS をパース・適用してから初めて `@font-face` を発見してフォントを取得する。これは LCP より遅いタイミング
- `preload` で HTML パース時にフォント取得を開始することで FOUT 時間を 100-300ms 短縮できる
- フォント取得は CORS 必須（`@font-face` は anonymous モード）なので preload も `crossorigin="anonymous"` が必須
- preload しすぎると優先度判定を壊すため、ファーストビューに必要な 1-2 ウェイトに限定する

**コード例**:
```tsx
// app/layout.tsx (Next.js)
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ja">
      <head>
        {/* Good: 必須ウェイトの woff2 を crossorigin 付きで preload */}
        <link
          rel="preload"
          as="font"
          type="font/woff2"
          href="/fonts/inter-regular.woff2"
          crossOrigin="anonymous"
        />
        <link
          rel="preload"
          as="font"
          type="font/woff2"
          href="/fonts/inter-bold.woff2"
          crossOrigin="anonymous"
        />
      </head>
      <body>{children}</body>
    </html>
  );
}

// Bad: crossorigin なし → 別リクエストとしてフォントが二重取得される
<link rel="preload" as="font" href="/fonts/main.woff2" />

// Bad: 全ウェイトを preload（多すぎて優先度を破壊）
<link rel="preload" as="font" href="/fonts/inter-100.woff2" crossOrigin="" />
<link rel="preload" as="font" href="/fonts/inter-200.woff2" crossOrigin="" />
{/* …500, 600, 700, 800, 900 も */}
```

**Next.js での自動化**:
```tsx
// next/font は preload を自動的に注入する
import { Inter } from 'next/font/google';

const inter = Inter({
  subsets: ['latin'],
  preload: true,          // デフォルト true。Next が自動で <link rel="preload"> を挿入
  weight: ['400', '700'], // 必要なウェイトに絞る
});
```

**preload の判断基準**:
- ファーストビューに含まれるテキストで使われるフォント・ウェイト
- 通常は 1-2 ウェイトに絞る（regular + bold 程度）
- アイコンフォントは preload せずに `font-display: block`（後述）

**出典**:
- [MDN: rel="preload"](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/preload) (MDN Web Docs)
- [web.dev: Preload web fonts](https://web.dev/articles/preload-optional-fonts) (web.dev)
- [Next.js Docs: next/font](https://nextjs.org/docs/app/building-your-application/optimizing/fonts) (Next.js 公式)

**バージョン**: HTML Living Standard / Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-16

---

### 3. `size-adjust` / fallback metrics でフォント差し替え時のレイアウトシフトを抑える

Web フォントとフォールバックフォントは字幅・行高が異なるため、差し替え時に CLS が発生する。
`@font-face` の `size-adjust` `ascent-override` `descent-override` `line-gap-override` でフォールバックを Web フォントの寸法に合わせる。

**根拠**:
- フォント差し替え時の CLS が Core Web Vitals の主要悪化要因の 1 つ
- フォールバックフォントを「Web フォントの形に近づけて」表示することで差し替え時の位置ずれをゼロに近づけられる
- Next.js の `next/font` は `adjustFontFallback` で自動的にこの計算を行う
- 手動計算は [Fontaine](https://github.com/unjs/fontaine) や [Capsize](https://seek-oss.github.io/capsize/) で支援できる

**コード例**:
```css
/* フォールバックフォントを Web フォントの寸法に合わせる */
@font-face {
  font-family: 'CustomFont-Fallback';
  src: local('Arial');
  size-adjust: 105%;             /* フォントサイズを 5% 拡大して幅を合わせる */
  ascent-override: 90%;          /* アセンダー（上余白）の高さ */
  descent-override: 22%;         /* ディセンダー（下余白）の高さ */
  line-gap-override: 0%;
}

body {
  /* Web フォントを最初に、フォールバックを次に */
  font-family: 'CustomFont', 'CustomFont-Fallback', sans-serif;
}

/* Web フォント本体 */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.woff2') format('woff2');
  font-display: swap;
}
```

**Next.js での自動化**:
```tsx
import { Inter } from 'next/font/google';

const inter = Inter({
  subsets: ['latin'],
  adjustFontFallback: true,  // デフォルト true。自動で size-adjust 等を計算
});

// next/font は CSS を自動生成する。手動の @font-face 不要
```

**Capsize での計算**:
```ts
import { createFontStack } from '@capsizecss/core';
import inter from '@capsizecss/metrics/inter';
import arial from '@capsizecss/metrics/arial';

const { fontFamily, fontFaces } = createFontStack([inter, arial]);
// 自動的に size-adjust 等を含む CSS を生成
```

**判定基準**:
- 主に **本文用フォント** で効果が大きい（見出しは行数が少なく影響軽微）
- 計算は 1 度だけ行えば良いため、ビルド時生成が望ましい
- 完璧な一致は困難でも、CLS を 0.05 → 0.001 等に劇的に改善できる

**出典**:
- [MDN: size-adjust](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face/size-adjust) (MDN Web Docs)
- [MDN: ascent-override](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face/ascent-override) (MDN Web Docs)
- [web.dev: Improved font fallbacks](https://developer.chrome.com/blog/font-fallbacks) (Chrome for Developers)
- [Capsize](https://seek-oss.github.io/capsize/) (SEEK OSS)

**バージョン**: Chrome 87+ (`size-adjust`), Firefox 89+, Safari 17+
**確信度**: 高
**最終更新**: 2026-05-16

---

### 4. woff2 を採用し、必要に応じてサブセット化する

Web フォントの配信フォーマットは woff2 一択。レガシー対応は不要。
日本語等の大規模文字セットフォントは `unicode-range` でサブセット化するか、可変フォント（variable font）の活用を検討する。

**根拠**:
- woff2 は全モダンブラウザ（IE 除く）でサポート済み。woff（v1）・ttf・otf より 30% 程度小さい
- ブラウザは `@font-face` 内の `src` リストを順に評価するため、woff2 を最初に書けば他フォーマットはダウンロードされない
- 日本語フォント（数 MB）はサブセット化で 30-100KB まで削減できる
- variable font は 1 ファイルで全ウェイト・斜体をカバーし、複数ウェイトをロードするより総容量が小さい

**コード例**:
```css
/* Good: woff2 のみ */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.woff2') format('woff2');
  font-display: swap;
}

/* Bad: 不要な複数フォーマット（容量とリクエスト数の浪費） */
@font-face {
  font-family: 'CustomFont';
  src:
    url('/fonts/custom.eot'),
    url('/fonts/custom.eot?#iefix') format('embedded-opentype'),
    url('/fonts/custom.woff2') format('woff2'),
    url('/fonts/custom.woff') format('woff'),
    url('/fonts/custom.ttf') format('truetype');
}
```

**サブセット化（日本語等）**:
```css
/* Latin 文字 */
@font-face {
  font-family: 'NotoSansJP';
  src: url('/fonts/noto-jp-latin.woff2') format('woff2');
  unicode-range: U+0000-00FF, U+0131, U+0152-0153, U+2000-206F;
  font-display: swap;
}

/* 日本語ひらがな・カタカナ */
@font-face {
  font-family: 'NotoSansJP';
  src: url('/fonts/noto-jp-kana.woff2') format('woff2');
  unicode-range: U+3000-30FF;
  font-display: swap;
}

/* 漢字（よく使う JIS 第一水準のみ等） */
@font-face {
  font-family: 'NotoSansJP';
  src: url('/fonts/noto-jp-kanji-common.woff2') format('woff2');
  unicode-range: U+4E00-9FFF;
  font-display: swap;
}
```

**Variable Font の活用**:
```css
/* Good: 1ファイルで全ウェイトをカバー */
@font-face {
  font-family: 'InterVariable';
  src: url('/fonts/inter-variable.woff2') format('woff2-variations');
  font-weight: 100 900;  /* 範囲指定 */
  font-style: normal;
  font-display: swap;
}

.thin   { font-weight: 100; }
.normal { font-weight: 400; }
.bold   { font-weight: 700; }
.black  { font-weight: 900; }

/* Bad: 各ウェイトを別ファイルでロード */
@font-face { font-family: 'Inter'; font-weight: 400; src: url('/fonts/inter-400.woff2'); }
@font-face { font-family: 'Inter'; font-weight: 500; src: url('/fonts/inter-500.woff2'); }
@font-face { font-family: 'Inter'; font-weight: 700; src: url('/fonts/inter-700.woff2'); }
{/* …全ウェイト分続く */}
```

**ツール**:
- [glyphhanger](https://github.com/zachleat/glyphhanger) — ページで実際に使われる文字を抽出してサブセット作成
- [pyftsubset](https://fonttools.readthedocs.io/en/latest/subset/) — Python の fontTools 経由でのサブセット化
- [Google Fonts](https://fonts.google.com/) — `&text=...` パラメータで動的サブセット URL を生成可能
- [Fontsource](https://fontsource.org/) — npm 経由で各ウェイト・サブセット済み woff2 を取得

**出典**:
- [MDN: @font-face - src](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face/src) (MDN Web Docs)
- [MDN: unicode-range](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face/unicode-range) (MDN Web Docs)
- [web.dev: Reduce web font size](https://web.dev/articles/reduce-webfont-size) (web.dev)
- [web.dev: Variable fonts](https://web.dev/articles/variable-fonts) (web.dev)

**バージョン**: woff2 は Chrome 36+, Firefox 39+, Safari 12+ / Variable Fonts は Chrome 62+, Firefox 62+, Safari 11+
**確信度**: 高
**最終更新**: 2026-05-16

---

## 関連プラクティス

- [`performance/core-web-vitals.md`](./core-web-vitals.md) - LCP・CLS の改善（フォント関連はここからリンク）
- [`web-standards/html.md`](../web-standards/html.md) - リソースヒント（`preload` 等）の使い分け
