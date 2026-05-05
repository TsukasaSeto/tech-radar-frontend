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

**根拠**:
- `sizes` を省略すると常に `100vw` として扱われ、不必要に大きな画像が配信される
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

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. OGP 画像は `next/og` で動的生成する

Open Graph 画像（OGP）はページごとに動的生成する必要がある場合、
`next/og`（ImageResponse API）を使いエッジで生成する。

**根拠**:
- エッジランタイムで実行されるためレイテンシが低い
- React コンポーネントの構文で OGP 画像をデザインできる
- 静的画像をすべてのページ分事前生成する必要がなくなる

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

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05
