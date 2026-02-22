# EPUB Reader Implementation Recipe (v3.5)

このドキュメントは、高機能な Electron ベースの EPUB リーダーを、生成系AIを用いてゼロから再構築するための完全かつ「責任境界を明確にした」技術仕様書です。

---

## 1. 開発環境とセキュリティ (Electron)
- **基盤**: Electron, Vanilla JavaScript, JSZip (EPUB解凍用)。
- **セキュリティ方針**: 
    - `contextIsolation: true`, `sandbox: true`, `nodeIntegration: false` を厳守。
    - ファイル操作は `ipcMain` / `ipcRenderer` を介し、`preload.js` でブリッジする。
- **UI基盤**: Flexbox を用いた 100vh 追従型レイアウト。`calc` に頼らず `flex: 1` で高さを自動分配する。

---

## 2. データ層：EPUB解析と BookModel
- **解析エンジン**: 外部ライブラリ（epub.js等）は使わず、`jszip` + `DOMParser` による自前パースを行う。
- **パース手順**:
    1. `META-INF/container.xml` から `full-path` を取得し OPF ファイルを特定。
    2. OPF 内の `manifest` でリソースをインデックス化し、`spine` で閲覧順序を確定させる。
- **画像抽出 (Comic用)**:
    - `spine` 順に HTML を走査し、`<img>` および `<svg><image>` を抽出。
    - SVG は `<img>` に変換し、パスを正規化して `data-rel-src` 属性に保持したフラットな `allImages` リストを作成する。
- **メモリ戦略**:
    - 全章の HTML 構造（DOM）は保持するが、**画像のバイナリは保持しない**。
    - 画像は描画直前に `URL.createObjectURL` で発行し、遷移時に必ず `revoke` する。

---

## 3. レンダリング・ストラテジー (Strategy Pattern)
Shadow DOM で隔離し、モードごとに描画戦略を切り替える。

### A. コミックモード (ComicStrategy)
- **表示単位**: 「1画像 = 1ページ」として `allImages` インデックスで管理。
- **見開きロジック**: 
    - `firstPageMode: "2"`（表紙あり）なら `offset = 1`。
    - `idx >= offset` は `(idx - offset) % 2 === 0` を起点に2枚並べる。
- **レイアウト**: `height: 100%; width: auto;` で高さを固定し、Flexbox で中央に密着させる。

### B. テキストモード (TextStrategy)
- **表示単位**: 全章を縦に連結した「一本の巻物」。
- **スタイル制限**: 外部CSS (`<link>`) は無視するが、インラインスタイルとセマンティックタグ (`ruby` 等) は保持する。
- **スクロール**: ブラウザ標準の縦スクロールに任せる。

---

## 4. ナビゲーションと進捗
- **進捗計算**: 
    - コミック: `currentIndex / totalImages`
    - テキスト: `currentSectionIndex / totalSections` （＋ `scrollTopRatio`）
- **レジューム機能**: 
    - キー: `location:${uniqueIdentifier || title}` （ISBN等を優先、なければタイトル）
    - 内容: `{ spineIndex, scrollTopRatio }`
- **目次ジャンプ**: パス解決は「完全一致 → 末尾一致 → ファイル名一致」の3段階で行う。

---

## 5. 設定項目 (Settings)
| キー名 | デフォルト | 内容 |
| :--- | :--- | :--- |
| `readingMode` | "comic" | comic / text |
| `spread` | "always" | 見開き設定 |
| `direction` | "rtl" | 開き方向 (ltr / rtl) |
| `theme` | "dark" | dark / light / sepia |
| `gap` | 0 | ページ間の隙間 |

---

## 6. FAQ: 実装の落とし穴と解決策 (Gotchas)

- **Q1. 画像が出ない**: 相対パス解決は「参照元HTMLのパス」を基準にせよ。
- **Q2. 固定レイアウト(FXL)**: `rendition:layout="pre-paginated"` の場合はテキストモードを無効化し、コミックモードを強制せよ。
- **Q3. 画面が白くなる**: `revokeObjectURL` は `img.onload` または `decode()` 完了後に行え。

---

## 7. AI への指示（プロンプト）のゴール
> 1. 「1pxの隙間もないコミック見開き」を実現せよ。
> 2. 「複雑なフォルダ階層でも目次が飛ぶ」ようパスを正規化せよ。
> 3. 「メモリ消費が増えない」よう、Blobのライフサイクルを徹底管理せよ。
> 4. 「最大化してもタスクバーに潜り込まない」Flexboxレイアウトを構築せよ。

---

## 8. 互換性と免責事項 (Support Policy & Disclaimer)
本プロジェクトの対応範囲および保証限界を以下の通り定義する。

**対応範囲 (In Scope):**
- EPUB 2.0 / 3.0 規格に準拠したリフロー型コンテンツ。
- 一般的なコミック EPUB（1ページ1画像の構成）。
- 固定レイアウト (Fixed Layout / FXL) コンテンツ（画像としてのレンダリング）。

**保証対象外 (Out of Scope):**
- **DRM (著作権保護)** が施されたコンテンツ。
- 高度な **JavaScript** や対話型スクリプトに依存したコンテンツ。
- 音声・動画の埋め込み、同期再生機能。
- 特定ベンダー（Amazon, Apple, Kobo等）の独自拡張仕様。
- 縦書きテキスト表示（本仕様では保守性の観点から横書きを標準とする）。

本仕様は「一般的な閲覧用途において、安定かつ軽量に動作すること」を最優先目標とする。

---
*Created on: 2026-02-21*
*Final Reconstruction Kit for Professional AI Development*
