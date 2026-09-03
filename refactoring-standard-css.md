# リファクタリング規準書 (CSS)

モダン標準 CSS（`@scope`, `@layer`, `@container`, CSS Custom Properties）を前提とする。

## 1. 基本方針

- CSS をスマートかつ読みやすい形にリファクタリングすることを目的とする。
- コンポーネント単位でスタイルが自己完結しており、配置場所に依存しない設計にする。
- コード全体の構造を見ただけで、コンポーネントの責務と依存関係が把握できるように整理する。

## 2. リファクタリング手順

- コード全体の構成を理解したうえで、
- 必要に応じてコンポーネント分割、責務の明確化を行い、
- 処理の簡素化（Custom Properties による抽象化、`@scope` によるカプセル化）や統合を実施してスマートな構成に変換し、
- 代替の全コードを提示し、コードのフローや設計意図を簡単に説明し、
- 曖昧な点がある場合は確認しつつ、基本的には意図を汲み取り進めてください。
- 全ての指示に合致しているか確認し、合致していない部分が見つかったら最初からやり直してください。
- コード実行結果（視覚的な見た目・レイアウト・アニメーション）が1mmでも変更されてしまう場合、リファクタリング失敗であるため、やり直してください。

## 3. モノ指向コンポーネント設計

CSS における「モノ」は**コンポーネント**です。コンポーネントとは「DOM 上に存在する、ひとつの意味ある実体」であり、kebab-case のクラス名で識別されます。

### クラス名は名詞にする

- コンポーネント名は必ず**名詞**にしてください。
  - OK: `.card`, `.badge`, `.avatar`, `.nav`, `.dialog`, `.user-card`
  - NG: `.highlight`, `.format`, `.animate`, `.process`（動詞）
- 動詞・動詞にもなり得る語で終わる名前は禁止です。
  - NG: `.formatter`, `.handler`, `.processor`, `.wrapper`, `.container`（機能を示唆する語）
- 抽象的で意味をなさない語は禁止です。
  - NG: `.common`, `.misc`, `.util`, `.helper`, `.manager`
- 名前は **2 単語以内**にしてください。3 単語以上は責務過多の兆候です。
  - NG: `.top-navigation-bar-item-container` → OK: `.nav-item`

### コンポーネントは「自分の見た目は自分が定義する」

スタイルはコンポーネントに**帰属させる**ことが原則です。コンポーネントの外から直接子要素を操作することは、オブジェクトの内部状態を外部から書き換えることと同義であり禁止です。

```css
/* NG: .page が .card の内部実装（span）を直接操作している */
.page .card span { color: red; }

/* OK: カスタムプロパティ経由で .card にだけ設定を注入する */
.page .card { --card-text: red; }
```

### グローバルユーティリティクラスは static メソッド相当——原則禁止

`.mt-4`, `.flex`, `.text-red` のようなグローバルユーティリティクラスは、オブジェクト指向でいう `static func` に相当します。

- **どのコンポーネントにも帰属しない**。HTML 側に散在し、管理境界が崩れる。
- **スタイルの責務が HTML に漏れ出す**。HTML が「どう見せるか」を知りすぎている。
- **コンポーネントの自己完結性（カプセル化）を破壊する**。

原則禁止。ただし以下は `@layer base` 内に限り例外として許容する（`static let` 相当）:

- ブラウザリセット・ノーマライズ
- `:root` に定義するデザイントークン（`--color-brand`, `--spacing-base` 等）
- グリッドやフレックスレイアウトの**親コンテナへの指定**（子の細部ではなく、構造の骨格のみ）

例外を使ってよいかの判断: 「このクラスはどのコンポーネントに帰属するか？」と問いかけ、答えられない場合は設計を見直す。

### 複数コンポーネントクラスの合成は禁止

- 1つの要素に複数のコンポーネントクラスを付けて見た目を合成しないでください（`class="card featured-box"` 等）。スタイルの多重継承にあたり、どちらの責務がその要素を支配するか一意でなくなります。
- バリアントは `data-*` 属性で、複数コンポーネントにまたがる共通の土台は `@layer base` のデザイントークンで表現してください。
- 例外は、フレームワークが複数クラスの付与を強制する場合のみです。

### 同一責務は集約する

- ページ上のまったく別の場所で使われていても、**責務が同じスタイルは1つのコンポーネントにまとめてください**。配置場所に縛られない名詞のコンポーネントとして切り出し、両方の場所でそのコンポーネントを使い回します。
- 責務が同じかどうかは、それぞれの役目を「〜を〜する」と一文で言い表し、その一文が一致するかで見ます。セレクタの形やプロパティ数は判断材料にしません。書き方が違っていても、まとめる際に一方へ揃えます。
- 例: 画面隅に出るトースト通知と本文中のインラインアラートは、役目がどちらも「一時的な知らせを目立たせて表示する」なので `.notice` に統合し、`data-variant` で見た目だけを分けます。コンポーネント名は `.toast` のように出現場所を冠さず、`.notice` のような一般名詞にします。
- まとめても責務は1つのままなので、状態バリアント数の目安（§4）には響きません。重複が消えるぶん総数はむしろ減ります。
- いずれ片方だけ別の役目に育ちそうなものは、1つのコンポーネントに押し込めないでください。ただし共通の土台を `@layer component` の基底クラスや共有カスタムプロパティとして切り出せるなら、そこは共有します。

## 4. コンポーネント構造制約

### 状態バリアント数（メソッド数に相当）——4〜5 個

1 つのコンポーネントが持つ**状態バリアント**（`@scope` 内のルールブロック数）は **4〜5 個を目安**にしてください。3 個未満は抽象化不足、6 個以上は責務過多の兆候です。

**状態バリアントのカウント対象:**
- `:scope`（デフォルト状態）
- `:scope:hover`, `:scope:focus-visible`, `:scope:active`（インタラクション状態）
- `:scope[data-variant="x"]`（スキンバリアント）
- `:scope[data-size="sm"]`（サイズバリアント）

```css
/* OK: 5 状態のバリアント（ちょうど良い複雑さ） */
@scope (.btn) {
  :scope             { /* 1. デフォルト */ }
  :scope:hover       { /* 2. ホバー    */ }
  :scope:focus-visible { /* 3. フォーカス */ }
  :scope[data-variant="primary"] { /* 4. プライマリ */ }
  :scope:disabled    { /* 5. 無効     */ }
}
```

**カウント対象外:**
- `@keyframes` 定義
- `:scope > .child` の単純な子要素配置指定（子コンポーネントの本体スタイルは子コンポーネント自身が持つ）
- `@container` ブロック（レイアウト文脈への適応であり、責務の増加ではない）

### ルールブロックの行数制限（25 行以下）

1 つのルールブロック（`{}` 内のプロパティ群）は **25 行以下**にしてください。超える場合は CSS Custom Properties でデフォルト値を集約するか、子コンポーネントへの責務移譲を検討してください。

### 公開カスタムプロパティ（公開インターフェース）

コンポーネントが外部に公開する CSS Custom Properties は `--component-property` の形式で定義し、コンポーネントのブロック先頭に列挙してください。これがコンポーネントの「公開インターフェース」です。

```css
@scope (.card) {
  :scope {
    /* 公開インターフェース（外部から上書き可能） */
    --card-bg: white;
    --card-radius: 8px;
    --card-padding: 1rem;
    --card-shadow: 0 2px 4px rgb(0 0 0 / 0.1);

    /* 内部実装（外部から上書きしない） */
    --_gap: calc(var(--card-padding) / 2);

    background: var(--card-bg);
    border-radius: var(--card-radius);
    padding: var(--card-padding);
    box-shadow: var(--card-shadow);
    gap: var(--_gap);
  }
}
```

### プライベートカスタムプロパティ

コンポーネント内部でのみ使うカスタムプロパティは `--_` プレフィックスで明示し、外部からの上書き対象でないことを示してください。

```css
/* Python の _ 接頭辞、Swift の private に相当 */
--_internal-gap: calc(var(--card-padding) / 2);
```

## 5. 命名規約

- コンポーネント名は **kebab-case**。クラス名の中に動詞を混ぜない。
- カスタムプロパティは `--component-property` の形式（コンポーネント名プレフィックス必須）。
  - OK: `--card-bg`, `--nav-height`, `--badge-color`
  - NG: `--color`, `--size`（スコープなし・抽象的すぎる）
- 状態は HTML の `data-*` 属性または擬似クラスで表現し、JS によるクラス付け外しを最小化する。
  - OK: `[data-state="open"]`, `:checked`, `:disabled`
  - NG: `.is-open`, `.active`, `.selected`（JS がクラス名文字列に依存しすぎる）
- デザイントークン（`:root` レベルの共有定数）は `--` のみのプレフィックスなしか、`--color-*`, `--spacing-*` のカテゴリプレフィックスを付ける（コンポーネント名は付けない）。
  - OK: `--color-brand`, `--spacing-base`, `--font-body`
  - NG: `--card-brand-color`（コンポーネントがグローバルトークン名を持つのは逆転）

## 6. カプセル化（@scope / Shadow DOM）

`@scope` を使い、スタイルをコンポーネント内部に閉じ込めてください。外部へのスタイル漏れや意図しない干渉を防ぎます。

```css
@scope (.card) {
  /* .card 内部にだけ適用される */
  :scope {
    background: var(--card-bg, white);
    border-radius: var(--card-radius, 8px);
  }
  /* .card 内の .title にだけ適用（外の .title には影響しない） */
  .title {
    font-size: 1.25rem;
  }
}
```

Web Components を使う場合は Shadow DOM によるカプセル化を優先してください。Shadow DOM 内のスタイルは外部から完全に隔離されます。`@scope` で十分な場合は Shadow DOM を強制しません。

## 7. 抽象化と多態性（CSS Custom Properties）

CSS Custom Properties は「インターフェース」です。コンポーネントの内部実装を隠蔽し、外部から設定値だけを注入できるようにしてください。

```css
/* コンポーネント定義（構造と公開インターフェース） */
@scope (.badge) {
  :scope {
    --badge-color: navy;
    --badge-bg: #e8f0fe;
    --badge-radius: 999px;
    --badge-size: 0.75rem;

    color: var(--badge-color);
    background: var(--badge-bg);
    border-radius: var(--badge-radius);
    font-size: var(--badge-size);
  }

  /* 多態性: 構造（セレクタ・プロパティ名）を変えず、値だけを差し替える */
  :scope[data-variant="danger"] {
    --badge-color: crimson;
    --badge-bg: #fce8e8;
  }

  :scope[data-variant="success"] {
    --badge-color: #1a5c2a;
    --badge-bg: #e8f5e9;
  }
}
```

「構造（HTML）」と「見た目（Skin）」の分離がこれで完結します。バリアントを追加するとき、セレクタ構造を壊さずに値の差し替えだけで済むことが良い設計の証拠です。

## 8. 文脈からの独立（@container）

コンポーネントは**自身が置かれたコンテナのサイズに自己適応**してください。`@media`（ビューポートサイズへの依存）は、コンポーネントをページ構造に結合させるため、コンポーネントレベルでは原則禁止です。

```css
/* コンポーネントの親をコンテナとして登録 */
.card-wrapper {
  container-type: inline-size;
  container-name: card;
}

@scope (.card) {
  :scope {
    display: flex;
    flex-direction: row;
  }
}

/* 配置場所ではなく、自分が置かれたコンテナサイズで自己変容 */
@container card (width < 400px) {
  @scope (.card) {
    :scope { flex-direction: column; }
  }
}
```

`@media` を使ってよいのは、ページ全体のレイアウト骨格（グリッドのカラム数等）を制御する**レイアウト層**に限ります。

## 9. 優先度の階層管理（@layer）

`@layer` でスタイルの責務階層を明示的に定義し、詳細度（Specificity）の衝突を排除してください。

```css
/* ファイル先頭でレイヤー順を宣言する（後が勝つ） */
@layer base, component, override;
```

| レイヤー | 役割 | 相当概念 |
|---|---|---|
| `base` | ブラウザリセット・HTML 要素のデフォルト整形・デザイントークン | フレームワーク基盤（Apple SDK 相当）|
| `component` | モノ指向コンポーネントのスタイル群 | インスタンスメソッド群 |
| `override` | テーマ・ページ固有の上書き（最小限） | Apple プロトコル準拠例外に相当 |

### 詳細度は上げない

- ID セレクタ（`#id`）は詳細度が爆発するため**禁止**。`data-*` 属性や `:scope` で代替する。
- `!important` はレイヤー構造の外に出るため**原則禁止**。`@layer` で解決できます。
- セレクタのネストを深くするほど詳細度が増す。ネストは `@scope` 内で最大 2 段まで。

## 10. 差分チェック対応

- リファクタリング前後で**視覚的な見た目が完全に一致する**ようにしてください。
- レイアウト・カラー・タイポグラフィ・アニメーション・レスポンシブ挙動をすべて確認してください。
- ビジュアルリグレッションテスト（Playwright, Chromatic 等）による確認を推奨します。
- リファクタリング中に発見した見た目の問題（デザインバグ）は**別 issue として起票**し、本リファクタリングでは変更しないでください。

## 11. Lint 対応

- **Stylelint** のデフォルトルールに適合してください。
- プロパティの記述順は以下の論理順序を推奨します:

```
1. CSS Custom Properties（--xxx）  ← 公開 → プライベートの順
2. レイアウト（display, position, grid, flex, gap）
3. ボックスモデル（width, height, padding, margin, border）
4. 視覚（color, background, box-shadow, opacity）
5. タイポグラフィ（font-*, line-height, text-*）
6. トランジション・アニメーション（transition, animation）
```

- `@layer`, `@scope`, `@container` を使用しない既存コードは、リファクタリング時に段階的に導入する。一度に全体を書き直さず、コンポーネント単位で移行する。

## 12. Jekyll / Liquid 例外規定

Jekyll は Liquid テンプレートエンジンを通じてビルド時に CSS を処理できる。§3「すべてのスタイルはコンポーネントに帰属させる」の原則は維持しつつ、以下を例外として定める。**例外はここに列挙したものに限る。**

### 12.1 Liquid 変数 vs CSS Custom Properties の使い分け

ビルド時（Jekyll がサイトを生成する瞬間）に決まる値と、ランタイム（ブラウザ上）で変わりうる値を明確に分離してください。

| 性質 | 手段 | 例 |
|---|---|---|
| ビルド時に確定する設定値 | Liquid 変数 `{{ site.data.theme.color }}` | ブランドカラー、フォントURL |
| ランタイムで変化しうる値 | CSS Custom Properties `var(--x)` | テーマ切替、ユーザー設定 |

```css
/* OK: ビルド時確定値は Liquid で注入し、ランタイム変化は Custom Properties で管理 */
:root {
  --color-brand: {{ site.data.theme.brand_color }};  /* Liquid → ビルド時固定 */
  --color-surface: var(--color-brand);               /* runtime で上書き可能 */
}
```

Liquid 変数と CSS Custom Properties を混在させることは許容するが、「どちらがビルド時でどちらがランタイムか」の境界を曖昧にしてはいけません。

### 12.2 フロントマター必須

Liquid タグ（`{{ }}`, `{% %}`）を含む CSS ファイルには、必ずファイル先頭に空のフロントマターを付けてください。フロントマターがないと Liquid タグが文字通り出力されます。

```css
---
---
/* このファイルは Jekyll が Liquid として処理する */
:root {
  --color-brand: {{ site.data.theme.brand_color | default: "#005fcc" }};
}
```

### 12.3 `_includes` とコンポーネントの 1 対 1 対応

Jekyll の `_includes/` ディレクトリのファイル名と、CSS コンポーネント名を一致させてください。これがモノ指向における「型名とファイル名の一致」に相当します。

```
_includes/card.html        ↔  @scope (.card) { }
_includes/user-card.html   ↔  @scope (.user-card) { }
_includes/nav-item.html    ↔  @scope (.nav-item) { }
```

Liquid include の名前（kebab-case）= CSS コンポーネント名（kebab-case）。これが崩れると責務の追跡が困難になります。

### 12.4 Liquid 条件分岐とスタイルの関係

Liquid の `{% if %}` によってクラス名が変化する場合、CSS 側では `data-*` 属性で対応してください。JS と同様に、クラス名文字列への依存を最小化します。

```liquid
{%- comment -%} NG: Liquid がクラス名を動的に変えている {%- endcomment -%}
<div class="badge {% if post.featured %}badge-featured{% endif %}">

{%- comment -%} OK: 状態は data-* 属性で渡し、CSS は data-* で分岐する {%- endcomment -%}
<div class="badge" {% if post.featured %}data-featured="true"{% endif %}>
```

```css
@scope (.badge) {
  :scope[data-featured="true"] {
    --badge-bg: gold;
  }
}
```

### 12.5 `url()` 内のパス

CSS の `url()` でサイト内アセットを参照する場合は `relative_url` フィルターを使用してください。パスをハードコードすると `baseurl` が異なる環境で壊れます。

```css
/* NG: パスをハードコード */
background-image: url('/assets/images/hero.webp');

/* OK: Liquid フィルターで環境非依存にする */
background-image: url('{{ "/assets/images/hero.webp" | relative_url }}');
```

ただしリファクタリングで `url()` のパスを変更することは §10（差分チェック）違反です。パスは既存のまま維持し、`relative_url` への移行は別 PR で行ってください。

### 12.6 Jekyll Sass パイプラインとの共存

Jekyll は Sass/SCSS をネイティブサポートします。Sass を併用する場合の境界を明確にしてください。

| 手段 | 解決タイミング | 用途 |
|---|---|---|
| Sass 変数（`$var`） | ビルド時（Sass コンパイル） | 中間計算・ループ生成 |
| Liquid 変数（`{{ }}`） | ビルド時（Jekyll 処理） | サイト設定値の注入 |
| CSS Custom Properties（`var(--x)`） | ランタイム（ブラウザ） | テーマ・状態変化 |

Sass 変数と CSS Custom Properties は目的が異なります。**ランタイムで変化しうる値を Sass 変数で管理してはいけません**（ビルド時に値が固定されるため）。

### 12.7 例外を使ってはいけないケース

- Liquid でコンポーネントのコアスタイル（色・サイズ等）をすべてビルド時注入し、CSS Custom Properties を使わないこと。テーマ切替やアクセシビリティ（ハイコントラスト）対応が不可能になる。
- `_includes` 名と CSS コンポーネント名を乖離させること。「どの HTML がどの CSS に対応するか」の追跡が困難になる。
- Liquid `{% for %}` ループで大量のユーティリティクラスを生成すること。§3 のグローバルユーティリティ禁止原則は Liquid 生成クラスにも適用される。
