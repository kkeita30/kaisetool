# 解説ボード作成ツール — 仕様書

## 概要

`tool/index.html` は、動画に合成する「解説ボード」画像（1920×1080px、背景透過PNG）をチャプターごとに書き出すためのスタンドアロンWebツールです。ビルドステップなし。ブラウザで開くだけで動作します。

---

## ファイル構成

```
tool/
├── index.html   # メインツール（React + Babel + html2canvas をCDN経由で使用）
├── theme.js     # デザイントークン（window.THEME として公開）
├── boards.json  # ボードデータ（チャプタータイトル＋テキスト行）
└── SPEC.md      # 本仕様書
```

フォントファイルは `tool/fonts/` フォルダから参照します。

```
tool/
└── fonts/
    ├── mgenplus-2cp-heavy.ttf
    ├── mgenplus-2cp-bold.ttf
    └── mgenplus-2cp-light.ttf
```

---

## boards.json — データ仕様

チャプターごとのボードデータを配列で格納します。

### スキーマ

```json
[
  {
    "title": "チャプタータイトル（string）",
    "lines": [
      "▼見出しテキスト",
      "⇒ポイントテキスト",
      "　字下げテキスト",
      "平文テキスト",
      ""
    ]
  }
]
```

### フィールド定義

| フィールド | 型 | 説明 |
|---|---|---|
| `title` | `string` | チャプタータイトル。ボード上部に表示される。 |
| `lines` | `string[] \| null` | テキスト行の配列。`null` の場合はタイトルのみ表示（まとめスライド等）。 |

### lines の行タイプ

各行文字列の**先頭1文字**でスタイルを判定します。

| 先頭文字 | タイプ | フォント | サイズ | 色 |
|---|---|---|---|---|
| `▼` | 見出し（midashi） | MgenHeavy | `midashiSize` | `midashiColor` |（表示は `midashiPrefix` で変更可）|
| `⇒` | ポイント（point） | MgenBold | `pointSize` | `pointColor`＋テキストストローク |（表示は `pointPrefix` で変更可）|
| `　`（全角スペース） | 字下げ（indent） | MgenLight | `indentSize` | `indentColor` |
| 空文字列 `""` | 空行（empty） | — | — | 高さ20pxのスペース |
| その他 | 平文（plain） | MgenLight | `plainSize` | `plainColor` |

---

## theme.js — デザイントークン仕様

`window.THEME` オブジェクトとして定義します。`index.html` から `<script src="theme.js">` で読み込まれます。

### 現在の値

```js
window.THEME = {
  // ボードサイズ
  boardW:       1920,
  boardH:       1080,

  // パネル配置（画面左上基準）
  panelLeft:    20,          // px
  panelTop:     20,          // px
  panelWidth:   920,         // px（画面の約半分）
  panelPadding: '24px 36px', // CSS shorthand

  // チャプタータイトル
  titleFont:    'MgenHeavy',
  titleSize:    50,          // px
  titleColor:   '#ffffff',

  // ▼ 見出し
  midashiFont:  'MgenHeavy',
  midashiSize:  50,
  midashiColor: '#f5b500', //オレンジ

  // ⇒ ポイント
  pointFont:    'MgenBold',
  pointSize:    50,
  pointColor:   '#ffffff',
  pointStroke:  '2px #043e80', // WebkitTextStroke（ブルーの縁取り）

  // 　字下げ
  indentFont:   'MgenLight',
  indentSize:   40,
  indentColor:  '#ffffff',

  // 平文
  plainFont:    'MgenLight',
  plainSize:    40,
  plainColor:   '#ffffff',

  // 行間（flexbox gap）
  lineGap:      6,           // px
};
```

### デザインの意図

- 背景は透過。動画編集ソフト側で映像の上に重ねて使用する。
- パネルは画面左半分（width 920px）を占有。右側に映像の被写体が映る想定。

---

## index.html — 機能仕様

### 起動方法

VS Code の **Live Server** 拡張を使用する。`index.html` を右クリック →「Open with Live Server」で起動。

> `boards.json` の自動読み込みに `fetch()` を使用しているため、`file://` プロトコル（ダブルクリックで直接開く）では動作しない。

### 操作フロー

1. 「boards.json」ファイル選択 → チャプター一覧がドロップダウンに展開
2. チャプターを選択 → タイトル・テキスト行がフォームに読み込まれる
3. フォームで内容を編集（行タイプの変更、テキスト編集、行の追加・削除・並び替え）
4. プレビューで確認（左側に1920×1080を0.5倍縮小表示）
5. 「PNG書き出し」ボタン → 1920×1080px の透過PNGをダウンロード

### PNG書き出しの仕組み

- html2canvas は CSS transform を正確にキャプチャできないため、
  画面外（`position: fixed; left: -9999px`）に **等倍（1920×1080）** の非表示ボードを配置し、
  そちらをキャプチャ対象とする。
- プレビュー用ボードは `transform: scale(0.5)` で縮小表示するが、キャプチャには使用しない。

### フォームの行タイプ選択肢

| 表示ラベル | value | 意味 |
|---|---|---|
| ▼ 見出し | `midashi` | 黄色・太字の見出し |
| ⇒ ポイント | `point` | 白地＋青縁取り |
| 　字下げ | `indent` | 細字・左インデント |
| 平文 | `plain` | 細字・通常 |
| 空行 | `empty` | 高さ20pxのスペース |

---

## 出力仕様

| 項目 | 値 |
|---|---|
| サイズ | 1920 × 1080 px |
| 形式 | PNG |
| 背景 | 透過（アルファチャンネルあり） |
| ファイル名 | `board_{タイトル}.png` |
| 用途 | 動画編集ソフトで映像レイヤーの上に重ねて合成 |

---

## AIへの補足

このツールを改修・拡張する際の注意点:

- **データの正規形は `boards.json`** です。
- **スタイルの変更は `theme.js` のみ**を編集します。`index.html` 内にハードコードされたスタイル値はありません。
- `index.html` はReact + JSX をBabelでトランスパイルしています（`<script type="text/babel">`）。ビルド不要ですが、ローカルファイルとしてブラウザで開く際にCORSの制約でフォントが読み込めない場合があります（その場合は簡易HTTPサーバーを使用）。
- html2canvas はCSS `transform` 内の要素を正確に処理できないため、キャプチャ用とプレビュー用のDOMを分けています（重要）。
