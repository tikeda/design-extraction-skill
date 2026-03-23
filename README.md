**日本語** | [English](./README.en.md)

# Design Extraction Skills for Claude Code

コードからUIを解析し、[Pencil](https://pencil.dev/)（.pen）ファイルに構造化されたデザインシステムを構築する Claude Code プラグインです。

これからのアプリケーション開発は、必ずしもデザインツールからスタートするとは限りません。コードから始まるプロジェクトが増え、エンジニアやPMなど非デザイナーがUIに関与する機会も増えていきます。それでもデザインの一貫性やクオリティは誰かがコントロールする必要があります。このプラグインは、そうした変化を理解し、新しい開発ワークフローの中でデザインを支えたいデザイナーに使ってもらいたいと思い作りました。

<img width="1772" height="1099" alt="sc" src="https://github.com/user-attachments/assets/421d4eda-455a-4199-b025-0f48d2cfb9a1" />

## Skills

### `/design-extraction:design-extraction`
**コード → デザインシステム・画面デザイン**

<table>
<tr>
<td width="50%">
<img height="auto" alt="スクリーンショット 2026-03-23 11 39 11" src="https://github.com/user-attachments/assets/b4eec655-d9e0-4f50-929e-dcf81cce1e53" /></td><td width="50%"><img width="1722" height="1125" alt="スクリーンショット 2026-03-23 9 52 19" src="https://github.com/user-attachments/assets/c830b7b3-4bad-4349-ad11-3acad3285283" /></td></tr>
</table>

コードベースを解析し、.pen ファイルにデザインシステムと画面デザインを再現します：

1. **デザイントークン** — カラー、フォント、スペーシング、角丸をコードから抽出 → variablesとして保存
2. **コンポーネント** — 繰り返し使われるUIパターンを特定 → 再利用可能なコンポーネントを作成
3. **不足要素** — 不足しているステート（hover、disabled、errorなど）を発見 → 追加を提案
4. **カタログ** — トークンとコンポーネントのビジュアル一覧を作成（実装済み vs. 提案）
5. **画面の再現** — メイン画面をコードから忠実に再現（オプションでスクリーンショットとの比較検証も可能）

### `/design-extraction:design-variants`
**コードの条件分岐 → 画面の状態パターン**

<table>
<tr>
<td width="50%">
<img width="794" height="595" alt="スクリーンショット 2026-03-23 12 15 36" src="https://github.com/user-attachments/assets/67681988-b5ec-4eef-8bc7-439b9a4beb87" /></td><td width="50%"><img width="1592" height="893" alt="image" src="https://github.com/user-attachments/assets/4612b436-d59e-4c4a-8a13-c8010077878a" /></td></tr>
</table>


`design-extraction` で作成したデザインシステムと画面をベースに、コードの条件分岐を解析して、画面の表示が変わるパターンをデザインとして生成します：

- **ユーザーロール** — ゲスト、無料ユーザー、有料ユーザー、管理者
- **データ状態** — ローディング、読み込み完了、空、エラー
- **機能の出し分け** — バッジの表示/非表示、セクションの有効/無効
- **インタラクション状態** — モーダルの開閉、アコーディオンの展開/折りたたみ

※複雑な条件がある画面のデザインデータをデザインツールで管理するのではなく、コードから必要な状態を都度生成してデザインする方が理にかなっているので、そのようなユースケースにも使えます。

## 対象プロジェクト

**小〜中規模のプロジェクト**で、コードがデザインの原本となっているケースに最適です：

- Vibe Codingやデザイナー以外が作成したプロジェクト
- 10〜30コンポーネント、数十画面程度
- 単機能アプリ、MVP、社内ツール、プロトタイプ

大規模・複雑なコードベースの場合は、対象ディレクトリやコンポーネントを絞って実行できます：

```bash
# 特定のディレクトリに絞る
/design-extraction:design-extraction --scope src/components/dashboard

# 会話の中で指定する
"Extract design system from the /src/views/settings/ directory only"
```

## 前提条件

- [Claude Code](https://claude.com/claude-code) がインストール済み
- [Pencil MCP server](https://pencil.dev/) が接続済み
- `.pen` ファイルをあらかじめリポジトリ内に保存しておき、Pencil エディタで開いている
- UIコードを含む対象プロジェクト（Vue、React、Svelte など任意のフレームワーク）

## インストール

### クイックスタート（単一セッション）

```bash
git clone https://github.com/tikeda/design-extraction-skill.git
claude --plugin-dir ./design-extraction-skill
```

### 恒久インストール

Claude Code セッション内で：

```
/plugin marketplace add tikeda/design-extraction-skill
/plugin install design-extraction@design-extraction-skill
```

## 使い方

```bash
/design-extraction:design-extraction
```

オプション：

```bash
# .pen ファイルを指定して抽出
/design-extraction:design-extraction design.pen
```

```bash
/design-extraction:design-variants
```

対象のViewファイルやバリアントは実行後にClaude Codeが一覧を表示するので、対話的に選択できます。直接指定することも可能です：

```bash
# Viewファイルを指定
/design-extraction:design-variants src/views/RecipeDetailView.vue

# .pen ファイルも指定
/design-extraction:design-variants src/views/RecipeDetailView.vue design.pen
```

## 仕組み

### Design Extraction（6 steps）

| ステップ | 内容 |
|------|------|
| 1 | リポジトリ解析（技術スタック、アセット、構成） |
| 2 | デザイントークン抽出 + 不足トークン追加 + トークンカタログ作成 |
| 3 | コンポーネント抽出（アトミック + 複合）→ コンポーネントカタログ構築 |
| 4 | デザインシステム監査 — 各コンポーネントの不足ステートを提案 |
| 5 | トークン ↔ コンポーネントのバインド |
| 6 | 画面の再現（スクリーンショット比較はオプション） |

### Design Variants（7 steps）

| ステップ | 内容 |
|------|------|
| 1 | 対象コンポーネントの特定 |
| 2 | 条件分岐の解析（`v-if`、三項演算子、computed など） |
| 3 | バリアントマトリクス構築 → ユーザーが生成対象を選択 |
| 4 | 各バリアントの差分仕様を作成 |
| 5 | デフォルト画面をコピー → 差分を適用 |
| 6 | トークンとコンポーネントのバインド検証 |
| 7 | コード条件アノテーション付きで検証・ラベリング |

## 設計思想

### Code First

コードが原本。スクリーンショットは概要確認の補助のみ。すべてのデザイン要素は実際のコードに紐づく必要があります。

### Images Are Images

コードが画像ファイルを使用している場合（`import logo from './logo.png'`）、.pen ファイルでも画像フィルとして配置します。テキストで代替しません。

### Tokens, Not Hardcoded Values

カラー、フォントサイズ、スペーシング、角丸はすべて `$variable-name` 参照を使用。ハードコードされた hex/px 値は画面作成後に検証・置換されます。

### Meaningful Variants, Not Explosion

条件分岐の全組み合わせを生成するのではなく、デザイナーが確認すべき意味のある状態だけを生成します。コードから必要な状態を都度生成する方が、すべてのパターンを事前に用意するより実用的です。

## .pen ファイル構成

```
document
├── Reference Screenshots       # （オプション）比較用のライブスクリーンショット
├── Token Catalog                # デザイントークン 🟢 実装済み / 🟠 提案
├── Component Catalog            # 全コンポーネント + ステート（単一の情報源）
│   ├── Button                   # Default🟢, Hover🟠, Focus🟠, Disabled🟠
│   ├── Card                     # Default🟢, Hover🟠
│   └── ...
├── Screens                      # 画面の再現
│   ├── Product List
│   ├── Product Detail
│   └── ...
└── Screen Variants              # 条件ステートバリアント
    ├── RecipeDetailView — Variants
    │   ├── Guest User
    │   ├── Loading
    │   ├── Error
    │   └── ...
    └── ...
```
