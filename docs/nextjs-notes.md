# Next.js 知見集

ハマったこと・注意事項を症状→原因→解決策の形式で記録する。
新しい知見は開発中に随時追記すること。

---

## レイアウト・スクロール

### `sticky` 要素が固定されずスクロールで流れていく

**症状**
`sticky top-0` を付けているのにヘッダーやフィルターバーがスクロール時に固定されない。

**原因**
`sticky` は最も近い「スクロールコンテナ」の中でのみ機能する。祖先要素に `overflow: hidden / auto / scroll` が設定されているとその要素がスクロールコンテナになり、`sticky` の基準が変わる。`html`/`body` 全体をスクロールさせている場合は `html`/`body` がコンテナになるため、中間の要素に `overflow-y-auto` があると機能しない。

**解決策**
`overflow-y-auto` を持つスクロールコンテナ（`<main>` 等）の中に `sticky` 要素を置く。`<main>` が独立したスクロールコンテナであれば `top-0` で確実に機能する。

```tsx
// layout.tsx
<AppHeader />         {/* main の外に置く */}
<main className="overflow-y-auto min-h-0 flex-1">
  {/* sticky 要素はここに置くと top-0 で固定される */}
  <div className="sticky top-0 z-10 bg-white">
    フィルターバーなど
  </div>
  {children}
</main>
```

---

### iOS の Dynamic Island / ノッチ下に sticky ヘッダーが隠れる

**症状**
iPhone で画面上部に sticky 固定した要素が Dynamic Island やノッチの下に隠れる。

**原因**
`top-0` は safe area を考慮しない。`<main>` がスクロールコンテナとして `<AppHeader>` の外にある設計の場合、`<AppHeader>` が safe area をすでに消費しているので sticky 要素は `top-0` で問題ない。一方、`<AppHeader>` と sticky 要素が同じスクロールコンテナ内にある場合は AppHeader の高さ分オフセットが必要。

**解決策**
sticky 要素が `<AppHeader>` と同じスクロールコンテナ内にある場合：

```tsx
<div
  className="sticky z-10 bg-white"
  style={{ top: 'calc(env(safe-area-inset-top) + 3.5rem)' }}
>
  {/* AppHeader の高さ（3.5rem = 56px）を加算 */}
</div>
```

`<main>` が `<AppHeader>` の外（兄弟要素）にある構成なら `top-0` で十分で `env(safe-area-inset-top)` の計算は不要。

---

### `flex` の子要素に `min-h-0` が必要

**症状**
`flex-col` の親の中で `overflow-y-auto` を付けた子要素がスクロールせず、コンテンツがはみ出す。

**原因**
flex アイテムのデフォルト `min-height` は `auto`（コンテンツサイズに依存）のため、明示的に `min-height: 0` を指定しないとコンテナを超えて広がる。

**解決策**
`overflow-y-auto` を持つ flex アイテムには必ず `min-h-0` を付ける。

```tsx
<div className="flex flex-col h-dvh">
  <AppHeader />
  <main className="overflow-y-auto min-h-0 flex-1">
    {children}
  </main>
</div>
```

---

## iOS Safari

### `<input>` / `<select>` にフォーカスするとページがズームされる

**症状**
フォーム入力欄をタップすると iOS Safari がページをズームイン・アウトし、フォーカスを外しても元に戻らないことがある。

**原因**
`font-size` が 16px 未満のフォーム要素にフォーカスすると iOS Safari がオートズームを発動する。

**解決策**
`<input>` / `<select>` / `<textarea>` の `font-size` は `text-base`（16px）以上にする。

```tsx
<input className="text-base ..." />   {/* OK */}
<input className="text-sm ..." />     {/* NG: 14px でズームが起きる */}
```

---

### ページに意図しない横スクロールが発生する

**症状**
コンテンツに横スクロールが発生し、ページが横にずれる。

**原因**
- flex コンテナ内の要素が想定外の幅を持つ
- `overflow-x: hidden` が `html`/`body` に設定されており、ピンチズームと干渉する

**解決策**

```css
/* globals.css */
html, body {
  overscroll-behavior-x: none; /* ビジュアルビューポートの横ドリフトを防止 */
}

body {
  /* overflow-x: hidden はスクロールコンテナを作り sticky を壊すことがある。
     clip はスクロールコンテナを作らないため sticky への影響が少ない。
     ピンチズームへの影響は実装依存で仕様上の保証はないが、
     実用上 clip の方が問題が少ないケースが多い。 */
  overflow-x: clip;
}
```

flex コンテナ内の `<Link>` は `block w-full` を付ける（`inline` だと iOS Safari で幅計算が狂う）。
`truncate` を使う要素には `min-w-0` を付ける（flex 内では明示しないと truncate が効かない）。

---

## PWA

### Next.js 14 以降で viewport が正しく設定されない

**症状**
iOS Safari でページが 980px のビューポートで描画され、ズームアウト状態になる。

**原因**
Next.js 14 以降、`<meta name="viewport">` は `metadata` オブジェクトに含めることが非推奨になり、`app/layout.tsx` から `viewport` オブジェクトとして別途エクスポートする必要がある。設定がないとブラウザのデフォルト（980px）が使われることがある。

**解決策**
`app/layout.tsx` に `viewport` をエクスポートする。

```typescript
import type { Viewport } from 'next'

export const viewport: Viewport = {
  width: 'device-width',
  initialScale: 1,
  // viewportFit: 'cover' は env(safe-area-inset-*) を実装しない限り設定しない
}
```

---

### `getUserMedia`（カメラ）が iOS Safari で失敗する

**症状**
バーコードスキャンや写真撮影のためにカメラを起動しようとすると iOS Safari でエラーになる。

**原因**
iOS Safari では `getUserMedia` をユーザー操作（タップ等）の同期コンテキストで呼ばないと許可が下りない。非同期処理（`await`、`setTimeout` 等）を挟んで遅延させると失敗する。

**解決策**
ボタンのクリックハンドラ内で `getUserMedia` を直接呼ぶ。ダイアログやモーダルを経由する場合は、ダイアログを開く前にストリームを取得してから渡す。

```typescript
// OK: クリックの同期コンテキストで getUserMedia を呼ぶ
async function handleOpenScanner() {
  const stream = await navigator.mediaDevices.getUserMedia({ video: true })
  setStream(stream)   // 取得してから state に保存してダイアログを開く
  setDialogOpen(true)
}
```
