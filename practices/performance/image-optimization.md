# 画像最適化のベストプラクティス

## ルール

### 1. すべての画像に `next/image` コンポーネントを使う

`<img>` タグの代わりに Next.js の `<Image>` コンポーネントを使う。
自動的なフォーマット変換・リサイズ・遅延読み込みが適用される。

**根拠**:
- WebP / AVIF への自動変換でファイルサイズを 30〜50% 削減できる
- ビューポートに基づいた `srcset` で適切なサイズの画像が配信される
- デフォルトで遅延読み込み（loading="lazy"）が適用される

**コード例**:
```tsx
import Image from 'next/image';

// Bad: 生の img タグ（最適化なし）
<img src="/hero.png" alt="ヒーロー" />

// Good: 固定サイズの画像
<Image
  src="/avatar.jpg"
  alt="ユーザーアバター"
  width={64}
  height={64}
  className="rounded-full"
/>

// Good: レスポンシブ画像（sizes で各ブレークポイントの表示幅を指定）
<Image
  src="/hero.jpg"
  alt="ヒーロー"
  width={1200}
  height={630}
  sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
  priority  // ファーストビューの画像は priority を付ける
/>

// Good: fill で親要素を埋める（aspect-ratio と組み合わせる）
<div className="relative aspect-video">
  <Image
    src="/thumbnail.jpg"
    alt="サムネイル"
    fill
    sizes="(max-width: 768px) 100vw, 50vw"
    style={{ objectFit: 'cover' }}
  />
</div>

// next.config.ts: 外部画像ドメインを許可
const nextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'images.example.com',
        pathname: '/uploads/**',
      },
    ],
  },
};
```

**出典**:
- [Next.js Docs: Image Optimization](https://nextjs.org/docs/app/building-your-application/optimizing/images) (Next.js公式)
- [Next.js Docs: Image Component API](https://nextjs.org/docs/app/api-reference/components/image) (Next.js公式)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. `sizes` 属性を正確に指定してレスポンシブ画像を最適化する

`next/image` の `sizes` 属性はブラウザが適切な解像度の画像を選択するために使う。
不正確な `sizes` は必要以上に大きな画像をダウンロードさせる。
`display:none` で隠す重複画像（PC 用 / モバイル用の二重配置）は、非表示側に最小の派生を選ばせる `sizes` を渡し、LCP 画像から `loading="lazy"` を外す前に非表示コピーの有無を確認する。

**根拠**:
- `sizes` を省略すると常に `100vw` として扱われ、不必要に大きな画像が配信される
- `loading="lazy"` は `display:none` の画像の読み込みも抑止していた。LCP 画像を eager にすると非表示側の複製も（大きなサイズで）読み込まれるため、非表示側には最小の派生を選ばせる `sizes` を渡す
- 正確な `sizes` によって、モバイルでは小さな画像が配信されデータ転送量が削減される
- レスポンシブレイアウトに合わせた `sizes` の記述が LCP 改善に直結する

**コード例**:
```tsx
// Bad: sizes を省略（常に全幅として扱われる）
<Image src="/card.jpg" alt="カード" width={400} height={300} />

// Good: 実際の表示幅に合わせた sizes を指定

// 2カラムグリッド（モバイルは全幅、デスクトップは半幅）
<Image
  src="/card.jpg"
  alt="カード"
  width={600}
  height={400}
  sizes="(max-width: 768px) 100vw, 50vw"
/>

// 3カラムグリッド（モバイルは全幅、タブレットは半幅、デスクトップは1/3幅）
<Image
  src="/product.jpg"
  alt="商品"
  width={400}
  height={400}
  sizes="(max-width: 640px) 100vw, (max-width: 1024px) 50vw, 33vw"
/>

// サイドバー付きレイアウト（コンテンツエリアは240px固定のサイドバーを除いた幅）
<Image
  src="/article.jpg"
  alt="記事"
  fill
  sizes="(max-width: 768px) 100vw, calc(100vw - 240px)"
/>
```

**出典**:
- [web.dev: Responsive images](https://web.dev/responsive-images/) (web.dev)
- [LCP の画像を eager にしたら、隠れている側の画像も読み込まれた](https://zenn.dev/kodomo_news/articles/243-eager-lcp-image-loads-hidden-twin) (Zenn、非表示複製と lazy の関係) ※2026-10-16 fetch
  > "これは `loading="lazy"` が付いていたから成り立っていたことで、eager にすると `display:none` の側の画像も読み込まれます。" (セクション "外れた前提")

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-10-16

---

### 3. OGP 画像は `next/og` で動的生成する

Open Graph 画像（OGP）はページごとに動的生成する必要がある場合、
`next/og`（ImageResponse API）を使いエッジで生成する。なお Node.js ランタイムの `ImageResponse` 実装は、Next.js `>=16.2.0 <16.3.6` で Critical の RCE（GHSA-vcvr-r3jv-pc5j）の影響を受けるため、16.x では 16.3.6 以降に上げる（15.x は RCE の対象外で 15.5.26 は hardening のみ。Edge の `ImageResponse` 実装は影響を受けない）。

**根拠**:
- エッジランタイムで実行されるためレイテンシが低い
- React コンポーネントの構文で OGP 画像をデザインできる
- 静的画像をすべてのページ分事前生成する必要がなくなる
- Node.js `ImageResponse` は Satori が生成する SVG のエスケープ不備に起因する RCE の影響を受けた（16.3.6 で上流依存の更新により修正）。16.2.0〜16.3.5 の Node.js ランタイムでこの API を使っている場合は 16.3.6 以降へのアップグレードを優先する

**コード例**:
```tsx
// app/og/route.tsx
import { ImageResponse } from 'next/og';
import { NextRequest } from 'next/server';

export const runtime = 'edge';

export async function GET(req: NextRequest) {
  const { searchParams } = new URL(req.url);
  const title = searchParams.get('title') ?? 'デフォルトタイトル';
  const description = searchParams.get('description') ?? '';

  return new ImageResponse(
    (
      <div
        style={{
          display: 'flex',
          flexDirection: 'column',
          width: '100%',
          height: '100%',
          padding: '60px',
          background: 'linear-gradient(to bottom right, #1a1a2e, #16213e)',
          color: 'white',
        }}
      >
        <h1 style={{ fontSize: 60, margin: 0 }}>{title}</h1>
        <p style={{ fontSize: 30, opacity: 0.8 }}>{description}</p>
      </div>
    ),
    { width: 1200, height: 630 }
  );
}

// app/blog/[slug]/page.tsx で OGP メタデータを設定
export async function generateMetadata({ params }: { params: { slug: string } }) {
  const post = await fetchPost(params.slug);
  const ogUrl = new URL('/og', process.env.NEXT_PUBLIC_APP_URL);
  ogUrl.searchParams.set('title', post.title);
  ogUrl.searchParams.set('description', post.excerpt);

  return {
    openGraph: {
      images: [{ url: ogUrl.toString(), width: 1200, height: 630 }],
    },
  };
}
```

**出典**:
- [Next.js Docs: OG Image Generation](https://nextjs.org/docs/app/building-your-application/optimizing/metadata#dynamic-image-generation) (Next.js公式)
- [Next.js Security Update for a Critical Upstream Issue](https://nextjs.org/blog/nextjs-security-update-september-22-2026) (Next.js公式ブログ、Node.js `ImageResponse` のRCE脆弱性GHSA-vcvr-r3jv-pc5jと修正版16.3.6/15.5.26) ※2026-09-22に実際にfetch成功

> "The issue affects the Node.js `ImageResponse` implementation in `next/og`. Under specific conditions, improper escaping in SVG output generated by Satori could lead to remote code execution due to vulnerabilities in other upstream dependencies... Applications using the Edge `ImageResponse` implementation are not affected."
> ([Next.js Security Update for a Critical Upstream Issue](https://nextjs.org/blog/nextjs-security-update-september-22-2026), セクション "Impact") ※2026-09-22に実際にfetch成功

**バージョン**: Next.js 13+（Node.js ランタイムで `ImageResponse` を使う 16.x は 16.3.6+）
**確信度**: 高
**最終更新**: 2026-09-22

---

### 4. `<picture>` 要素と `srcset` でレスポンシブ画像を最適化する（Next.js 外の場合）

Next.js を使わない環境（静的HTML・その他フレームワーク）では `<picture>` 要素と
`srcset` 属性を使い、デバイスの解像度・ビューポートに最適な画像を配信する。

**根拠**:
- `srcset` によりブラウザが最適な解像度の画像を自動選択できる
- `<picture>` 要素でフォーマット対応状況に応じてAVIF / WebP / JPEG をフォールバックできる
- モバイル端末へのデータ転送量を大幅に削減でき、LCP の改善にも寄与する

**コード例**:
```html
<!-- Bad: 単一解像度の img タグ（すべてのデバイスに同じ画像） -->
<img src="/hero.jpg" alt="ヒーロー" />

<!-- Good: picture + srcset でフォーマットとサイズの両方を最適化 -->
<picture>
  <!-- AVIF: 最小サイズ、モダンブラウザが対応 -->
  <source
    type="image/avif"
    srcset="/hero-400.avif 400w, /hero-800.avif 800w, /hero-1200.avif 1200w"
    sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
  />
  <!-- WebP: AVIF非対応ブラウザ向けフォールバック -->
  <source
    type="image/webp"
    srcset="/hero-400.webp 400w, /hero-800.webp 800w, /hero-1200.webp 1200w"
    sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
  />
  <!-- JPEG: 最終フォールバック -->
  <img
    src="/hero-800.jpg"
    srcset="/hero-400.jpg 400w, /hero-800.jpg 800w, /hero-1200.jpg 1200w"
    sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
    alt="ヒーロー"
    width="1200"
    height="630"
    loading="lazy"
  />
</picture>

<!-- art direction: モバイルとデスクトップで異なる構図の画像を使い分ける -->
<picture>
  <source
    media="(max-width: 768px)"
    srcset="/hero-mobile.avif"
    type="image/avif"
  />
  <source
    media="(max-width: 768px)"
    srcset="/hero-mobile.webp"
    type="image/webp"
  />
  <img src="/hero-desktop.jpg" alt="ヒーロー" width="1200" height="630" />
</picture>
```

**出典**:
- [MDN: Responsive images](https://developer.mozilla.org/en-US/docs/Learn/HTML/Multimedia_and_embedding/Responsive_images) (MDN Web Docs)
- [MDN: The picture element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/picture) (MDN Web Docs)

**バージョン**: HTML5（全モダンブラウザ対応）
**確信度**: 高
**最終更新**: 2026-05-06

---

### 5. AVIF フォーマットを優先して使用する

画像フォーマットの選択では AVIF を第一候補とし、非対応ブラウザには WebP でフォールバックする。
JPEG / PNG を新規に使用するのは最終手段とする。

**根拠**:
- AVIF は JPEG 比で平均 50% 以上のファイルサイズ削減が見込める（WebP 比でも 20〜30%）
- Chrome・Firefox・Safari（16.4+）など主要ブラウザで対応済み（2024年時点でカバレッジ 90%+）
- ファイルサイズ削減はLCPの改善、通信コスト削減、Core Web Vitals スコア向上に直結する
- Next.js は `<Image>` コンポーネントで自動的に AVIF を配信するため追加設定不要

**コード例**:
```tsx
// Next.js: Image コンポーネントは自動的に AVIF を優先配信
import Image from 'next/image';

// AVIF 対応ブラウザには .avif、非対応には .webp、最終的には元フォーマットを配信
<Image
  src="/product.jpg"  // 元ファイルは JPEG でも Next.js が自動変換
  alt="商品"
  width={800}
  height={600}
/>

// next.config.ts: フォーマット優先順位のカスタマイズ
const nextConfig = {
  images: {
    formats: ['image/avif', 'image/webp'],  // デフォルト設定（AVIF優先）
    // AVIF は WebP より圧縮率が高いが、エンコードに時間がかかるため
    // トラフィックが多いサイトでは CDN でのキャッシュが重要
  },
};

// 静的サイト / 手動変換: squoosh CLI または sharp で AVIF 生成
// npx @squoosh/cli --avif '{}' --webp '{}' images/
```

```ts
// sharp を使ったビルド時の自動変換スクリプト（カスタムビルド環境向け）
import sharp from 'sharp';
import { glob } from 'glob';

const images = await glob('public/images/**/*.{jpg,jpeg,png}');

for (const imagePath of images) {
  const basePath = imagePath.replace(/\.(jpg|jpeg|png)$/, '');

  // AVIF 生成（品質 80 でも十分な画質）
  await sharp(imagePath).avif({ quality: 80 }).toFile(`${basePath}.avif`);

  // WebP フォールバック生成
  await sharp(imagePath).webp({ quality: 85 }).toFile(`${basePath}.webp`);
}
```

**出典**:
- [web.dev: Use AVIF images](https://web.dev/articles/avif) (web.dev / 2023)
- [MDN: AVIF image format](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types#avif_image) (MDN Web Docs)

**バージョン**: Next.js 13+（自動対応）、sharp 0.32+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 6. `images.minimumCacheTTL` のデフォルト変更を把握し、更新頻度の高い画像では明示的に設定する

Next.js 16.0.0 で `next/image`（`/_next/image` 経由で配信される最適化画像）の `minimumCacheTTL` デフォルト値が
60 秒から 14,400 秒（4時間）に変更された。OGP 画像のように差し替え頻度が高い画像は、
オリジンの `Cache-Control: max-age` が短くても `minimumCacheTTL` の値が下限として優先されるため、
アップデート後に反映まで最大 4 時間かかるようになるケースがある。

**根拠**:
- `minimumCacheTTL` は最適化済み画像をサーバー側でキャッシュする**最小秒数**であり、オリジンの `max-age` がこれより長い場合のみオリジン側が優先される（`max-age` が短くても `minimumCacheTTL` を下回ることはない）
- Next.js 16 でのデフォルト変更に気づかずアップグレードすると、SNS 等の OGP 画像プレビューが古いまま長時間残る事故につながる
- 対象画像の更新頻度に応じて `next.config.ts` で明示的に上書きするのが安全
- 典型的な影響ケースは OGP 画像を `/_next/image?url=...` 経由で配信している構成。`og:image` に `next/og` の Route Handler の URL を直接指定すれば `/_next/image` のキャッシュ層を通らず、`minimumCacheTTL` の影響を受けない

**コード例**:
```ts
// next.config.ts
const nextConfig = {
  images: {
    // OGP 画像など頻繁に差し替わる画像がある場合は明示的に短くする
    // （Next.js 16 のデフォルトは 14400 秒 = 4時間）
    minimumCacheTTL: 60,
  },
};
```

```tsx
// Bad: 差し替え頻度の高い OGP 画像を /_next/image 経由で配信する
// → Next.js 16 では minimumCacheTTL（既定 14,400 秒）が下限になり、最大 4 時間古い画像が配信される
export const metadata = {
  openGraph: {
    images: [`/_next/image?url=${encodeURIComponent("/og.png")}&w=1200&q=100`],
  },
};

// Good: next/og の Route Handler を直接指定する（/_next/image のキャッシュ層を通らない）
export const metadata = {
  openGraph: {
    images: ["/api/og"],
  },
};
```

**出典**:
- [Next.js 16 で OGP 画像が 4 時間キャッシュされた話と minimumCacheTTL 移行](https://zenn.dev/cosoado/articles/nextjs16-image-cache-ttl-4h-ogp-stale) (Zenn、`images.minimumCacheTTL` のデフォルト変更実測とOGP画像への影響、対策コード例) ※2026-09-14に実際にfetch成功

> "`minimumCacheTTL` はその変換済み画像をサーバー側でキャッシュする最小秒数。"
> ([Next.js 16 で OGP 画像が 4 時間キャッシュされた話と minimumCacheTTL 移行](https://zenn.dev/cosoado/articles/nextjs16-image-cache-ttl-4h-ogp-stale), セクション "minimumCacheTTL とは") ※2026-09-14に実際にfetch成功

> "`minimumCacheTTL` は「最小値」なのでオリジンが長い `Cache-Control: max-age` を返す場合はそちらが優先される。"
> ([Next.js 16 で OGP 画像が 4 時間キャッシュされた話と minimumCacheTTL 移行](https://zenn.dev/cosoado/articles/nextjs16-image-cache-ttl-4h-ogp-stale), セクション "対策") ※2026-09-14に実際にfetch成功

**バージョン**: Next.js 16+（デフォルト値変更）
**確信度**: 中
**最終更新**: 2026-09-15

---

### 7. `images.remotePatterns` は必要最小限のホスト・パスに絞り、Next.js を最新パッチ（16.3.8 / 15.5.27 以降）に保つ

`next/image` の Image Optimization は `images.remotePatterns` に許可した外部 URL をサーバー側で取得して最適化する。
このため、許可リストに載ったホストが攻撃者に制御され得る場合（ユーザー投稿コンテンツのホスト、ワイルドカード許可など）、プライベート IP 帯へのリクエストを誘発する SSRF の経路になる。
Next.js 2026-09 セキュリティリリース（CVE-2026-94483、High）で修正済みのため、`remotePatterns` を使っているアプリは必ずパッチを適用し、あわせてパターンを最小化する。

**根拠**:
- 公式アドバイザリは「許可リスト済みの攻撃者制御 URL が Image Optimization 中の SSRF（例: プライベート IP 帯）につながる」と説明している
- `images.remotePatterns` を設定していなければ影響を受けないため、許可範囲を狭めること自体が攻撃面の縮小になる
- 修正版は v16.3.8（Active LTS）と v15.5.27（Maintenance LTS）

**コード例**:
```ts
// Good: プロトコル・ホスト名・パスを具体的に指定する
const nextConfig = {
  images: {
    remotePatterns: [
      { protocol: 'https', hostname: 'images.example.com', pathname: '/uploads/**' },
    ],
  },
};
```

```ts
// Bad: ワイルドカードで広く許可し、攻撃者が制御できるホストまで含めてしまう
const nextConfig = {
  images: {
    remotePatterns: [{ protocol: 'https', hostname: '**' }],
  },
};
```

```bash
npm install next@16.3.8   # 16.3 系
npm install next@15.5.27  # 15.5 系
```

**アンチパターン**:
- `hostname: '**'` など過度に広い許可で外部画像を配信する
- 月次セキュリティリリースのパッチを適用せず、`remotePatterns` の許可範囲だけに防御を依存する

**出典引用**:
> "An attacker-controlled, allow-listed remote URL can lead to Server-Side Request Forgery (for example to private IP ranges) during Image Optimization. If no `images.remotePatterns` are configured, your application is not affected."
> ([September 2026 Security Release](https://nextjs.org/blog/september-2026-security-release), セクション "Server-Side Request Forgery in Image Optimization (High Severity)") ※2026-09-30に実際にfetch成功

**出典**:
- [September 2026 Security Release](https://nextjs.org/blog/september-2026-security-release) (Next.js 公式ブログ) ※2026-09-30に実際にfetch成功

**バージョン**: Next.js 16.3.8+ / 15.5.27+
**確信度**: 高（公式ブログ）
**最終更新**: 2026-09-30

---
