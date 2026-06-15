# UI 知見集

Webアプリのモバイル UI でハマったこと・注意事項を症状→原因→解決策の形式で記録する。
新しい知見は開発中に随時追記すること。

---

## タッチ・操作性

### スマホでボタン・入力欄が小さくてタップしにくい

**症状**
スマホでボタンや入力欄を正確にタップできない。特にアイコンボタンがタップしにくい。

**原因**
デフォルトのコンポーネントサイズが PC 向けに設計されており、モバイルのタッチ推奨サイズ（44px）を下回っている。

**解決策**
個別ページで上書きするのではなく、ベースコンポーネント（`Button`・`Input`・`Select` 等）のデフォルトサイズを 44px に設定する。漏れを防げる。

```tsx
// Button のデフォルト高さを h-11（44px）に設定
const buttonVariants = cva("...", {
  variants: {
    size: {
      default: "h-11 px-4",   // 44px
      sm: "h-9 px-3",
      lg: "h-12 px-5",
      icon: "size-11",        // アイコンボタンも 44px
    },
  },
})

// Input も同様
<input className="h-11 ..." />
```

---

### フィルターチップの横スクロールがズームと誤認される

**症状**
チップが 1 行に収まらないとき横スクロールを実装したところ、iOS Safari がページのズーム状態と誤認してページ全体がずれることがある。

**原因**
`overflow-x: auto` による横スクロールコンテナが iOS Safari のビジュアルビューポート判定と干渉する。

**解決策**
`flex-wrap` で折り返す。チップが 1 行に収まるよう `px-2`・`gap-1.5` でコンパクト化する。

```tsx
{/* NG: 横スクロール */}
<div className="flex gap-2 overflow-x-auto">
  <button className="shrink-0 px-3 ...">チップ</button>
</div>

{/* OK: 折り返し */}
<div className="flex flex-wrap gap-1.5">
  <button className="px-2 ...">チップ</button>
</div>
```

---

## ダイアログ・モーダル

### アニメーションつき Dialog の中の `useRef` が null になる

**症状**
Dialog を開いた直後の `useEffect` で `videoRef.current` などが `null` のままになり、初期化処理が走らない（例: カメラ映像が黒いまま）。

**原因**
`@base-ui/react` など一部の Headless UI ライブラリは Dialog の children をアニメーションと共に遅延マウントする。`open=true` 時点ではまだ DOM に要素が存在しない。

**解決策**
`useRef` の代わりに callback ref + `useState` を使い、要素のマウントを検知してから `useEffect` を実行する。

```tsx
// NG: open 直後は null になる
const videoRef = useRef<HTMLVideoElement | null>(null)
useEffect(() => {
  if (!videoRef.current) return
  // ...
}, [open])

// OK: callback ref でマウントを検知
const [videoEl, setVideoEl] = useState<HTMLVideoElement | null>(null)
const videoRefCallback = useCallback((el: HTMLVideoElement | null) => {
  setVideoEl(el)
}, [])
useEffect(() => {
  if (!videoEl) return  // マウントされてから実行される
  // ...
}, [videoEl])

<video ref={videoRefCallback} />
```

---

### Dialog が重複してレンダリングされる

**症状**
Dialog（Tips・確認モーダル等）が 2 重に表示される。

**原因**
JSX 内の複数箇所に同一の Dialog コンポーネントを配置してしまっている。

**解決策**
Dialog の配置は 1 か所に限定する。grep で重複がないか確認する。

```bash
grep -rn "SomeDialog" src/
```

---

### スマホで Dialog のコンテンツが長いと閉じられなくなる

**症状**
Dialog を下までスクロールした後、×ボタンが見えない位置にあり閉じられない。

**原因**
×ボタンがヘッダー固定の場合、コンテンツが長いとスクロール後に画面外に出る。

**解決策**
コンテンツ末尾に全幅の「閉じる」ボタンを追加する。

```tsx
<DialogContent>
  {/* ... コンテンツ ... */}
  <button
    onClick={() => setOpen(false)}
    className="mt-4 w-full rounded-lg bg-gray-100 py-3 text-sm"
  >
    閉じる
  </button>
</DialogContent>
```

---

### `Tabs` コンポーネントが `sticky` 要素を妨害する

**症状**
`sticky` で固定したフィルターバーがスクロール時に固定されない。`Tabs` コンポーネントに内包しているときだけ発生する。

**原因**
`Tabs` / `TabsList` が内部で flex コンテキストを作るため、その中の `sticky` 要素のスクロールコンテナが変わる。

**解決策**
タブの代わりにチップボタン UI（`button` + `onClick`）に置き換える。`sticky` にしたい要素は Tabs の外に出す。

```tsx
{/* NG: Tabs 内の sticky は不安定 */}
<Tabs>
  <div className="sticky top-0">...</div>
  <TabsContent>...</TabsContent>
</Tabs>

{/* OK: Tabs を使わずチップボタン */}
<div className="sticky top-0">
  {OPTIONS.map(opt => (
    <button key={opt} onClick={() => setFilter(opt)}>{opt}</button>
  ))}
</div>
```

---

## フォーム・入力

### ラベル横の補助リンクがスマホで右端に切れる

**症状**
`<label>` の横に「保存方法の目安を見る」などの補助リンクを配置すると、スマホ幅で右端が見切れる。

**原因**
ラベルと補助リンクを横並びにしたとき、コンテンツが画面幅に収まらない。

**解決策**
補助リンクは入力欄の直下に配置する。

```tsx
{/* NG: ラベル横に配置 */}
<div className="flex justify-between">
  <label>賞味期限</label>
  <a href="/tips">保存方法の目安</a>  {/* スマホで切れる */}
</div>

{/* OK: 入力欄の下に配置 */}
<label>賞味期限</label>
<input type="date" />
<a href="/tips" className="text-sm text-blue-600">保存方法の目安</a>
```

---

## サードパーティ UI ライブラリ

### `@base-ui/react` の `Select.Value` が value（ID/英語）のまま表示される

**症状**
`SelectItem` に日本語ラベルを設定しているのに、`SelectValue` には生の value（ID や英語キー）が表示される。

**原因**
`@base-ui/react` の `SelectValue` は `SelectItem` の `children` を自動継承しない（shadcn/ui とは挙動が異なる）。

**解決策**
`SelectValue` に render prop を渡してラベルを明示する。

```tsx
{/* NG: value がそのまま表示される */}
<SelectValue />

{/* OK: render prop でラベルに変換 */}
<SelectValue>
  {(v: StorageLocation | null) =>
    v ? STORAGE_LOCATION_LABELS[v] : "選択してください"
  }
</SelectValue>
```

---

## カメラ・メディア

### `html5-qrcode` が iOS Safari でカメラ起動直後に停止する

**症状**
バーコードスキャン画面を開くとカメラが一瞬起動してすぐ停止し、スキャンできない。

**原因**
`html5-qrcode` は iOS Safari で「カメラ起動 → 即停止 → エラー」になる既知問題が複数報告されており、設定変更では回避困難。

**解決策**
`@zxing/browser` + `@zxing/library` に置き換える。

```bash
npm uninstall html5-qrcode
npm install @zxing/browser @zxing/library
```

```tsx
import { BrowserMultiFormatReader } from '@zxing/browser'

const reader = new BrowserMultiFormatReader()
const controls = await reader.decodeFromConstraints(
  { video: { facingMode: 'environment' } },
  videoElement,
  (result) => { if (result) onScan(result.getText()) }
)
return () => controls.stop()  // クリーンアップ
```

---

### iOS Safari で `<video>` の映像が表示されない

**症状**
カメラやストリーム映像を `<video>` 要素に表示しようとしても iOS Safari では黒画面のまま。

**原因**
iOS Safari は `playsInline` 属性がないと `<video>` をフルスクリーン再生しようとし、インライン表示されない。

**解決策**
`<video>` に `playsInline`・`muted`・`autoPlay` を明示する。

```tsx
<video
  playsInline  {/* iOS Safari でインライン表示するために必須 */}
  muted
  autoPlay
  ref={videoRefCallback}
  className="w-full"
/>
```

---

## PWA・アイコン

### iOS ホーム画面アイコンに白余白がついて小さく見える

**症状**
`apple-touch-icon` を設定したが、iOS ホーム画面で他アプリより小さく見える。アイコン周囲に白い余白がある。

**原因**
画像にパディングが含まれており、実際のデザインが縮小されている。

**解決策**
`apple-touch-icon` は余白なし（full-bleed）デザインにする。背景色を透明にせず、端まで埋める。角丸は不要（iOS が自動で適用する）。

```
推奨サイズ: 180×180px
余白: なし（full-bleed）
角丸: 不要
```

Next.js で `src/app/apple-icon.tsx` を `ImageResponse` で動的生成する場合も、`padding` を 0 にして端まで描画する。
