# 第8章：ツールとスキル

> _「ドラえもん、なんか出してよ！」——のび太の無茶振りに、ドラえもんは四次元ポケットから「道具」を取り出す。しかし、ドラえもんの真の強みは道具そのものではない。**どの道具を、いつ、どう使うかを知っている**ことだ。OpenClawも同じである。26個のコアツールという「道具」と、約50個の組み込みスキルという「使い方の知識」。この2つが揃って初めて、ただの会話AIが「行動するエージェント」に変わるのだ。_

---

前章でチャンネルという「声」を手に入れたOpenClaw。Discord、Telegram、WhatsAppから話しかけられるようになった。だが、会話だけでは物足りない。ファイルを読み書きし、シェルコマンドを実行し、Webを検索し、画像を生成する——エージェントが実世界で「行動」するための仕組みが、この章のテーマだ。

## この章が解き明かす「行動力の正体」

ツールとスキル。この2つの言葉は似ているようで、まったく異なる概念である。混同したままだと、OpenClawの設計思想が見えてこない。

この章を読み終える頃には、以下のことが明確になるだろう：

- **ツール**（26個のコアツール）の全体像と11カテゴリの分類体系
- **スキル**（約50個の組み込みスキル）の仕組みと発動メカニズム
- ツールとスキルの根本的な違い——**アクション vs 知識注入**
- **ツールポリシー**による6層のセキュリティ制御
- **カスタムスキル**の作り方——SKILL.md一つで専門家を生み出す方法
- **プラグイン**によるツール拡張と、スキルとの本質的な違い

核心を先に言おう。**ツールは「筋肉」、スキルは「技術書」だ。** 筋肉だけでは何もできないし、技術書だけでは動けない。両方揃って初めてエージェントは専門家になる。

<!-- IMAGE: ch08-tools-vs-skills.png - ツール（アクション/関数呼び出し）とスキル（知識注入/プロンプト）の違いを対比した概念図 -->

---

## 8.1 ツールとスキル — エージェントの「手足」と「技術書」

まずは全体像を掴もう。

### ツール＝エージェントが実行できるアクション

ツールとは、エージェントが呼び出せる**関数**のことだ。JSONSchema（パラメータの型や必須/任意をJSON形式で定義する仕組み）で定義されたパラメータを受け取り、処理を実行し、結果を返す。ファイルの読み書き、シェルコマンドの実行、Web検索——これらはすべてツールとして実装されている。

たとえば `exec` ツールは「シェルコマンドを実行する」というアクション。`read` ツールは「ファイルの中身を読む」というアクション。ツールは「何ができるか」の一覧であり、エージェントの**筋肉**にあたる。

### スキル＝エージェントに手順知識を与えるプロンプト注入

一方、スキルはアクションではない。スキルとは、エージェントのシステムプロンプトに**手順知識を注入する**仕組みだ。「天気予報を取得するには、`exec` ツールで `curl wttr.in` を実行せよ」——こうした専門的な手順書がスキルの正体である。

スキル自体は何も「実行」しない。エージェントに「やり方」を教え、エージェントが既存のツールを使って実行する。スキルは**技術書**なのだ。

### 連携パターン：ユーザーの質問から結果返却まで

ツールとスキルがどう連携するか、具体的な流れを見てみよう：

```
[ユーザー] 「今日の天気は？」
      ↓
[スキルのdescriptionでマッチ] → weatherスキルが該当
      ↓
[エージェントがSKILL.mdを読む] → wttr.inの叩き方を学ぶ
      ↓
[SKILL.mdの手順に従い、execツールを呼び出す]
      ↓
[exec実行: curl wttr.in/Tokyo] → 天気情報を取得
      ↓
[ユーザーに結果を返す] 「東京は晴れ、気温22℃です」
```

ここが重要なポイントだ。`exec` ツール（筋肉）は誰でも持っている。しかし、`weather` スキル（技術書）を読んで初めて「天気予報の取り方」がわかるのである。ツール単体では「何でもできるが何もわからない」状態。スキルが加わって初めて「専門家」になる。

> 💬 **こうじの実感：** 最初は「ツールもスキルも同じじゃね？」って思ってたんだけど、実際に自分でスキルを作ってみて違いがハッキリわかった。execツールだけだと「何でもできるけど何もわからない」状態。そこにSKILL.mdを1枚置くだけで、急に専門家になる。この分離が美しいんだよな。

---

## 8.2 コアツール全26種 — 11カテゴリの武器庫

OpenClawは `CORE_TOOL_DEFINITIONS` で定義された**26個のコアツール**を持つ。これらは11のセクション（カテゴリ）に分類されている。全体像を一気に見てみよう。

<!-- IMAGE: ch08-tool-catalog-map.png - 11カテゴリに分類された26個のコアツールの一覧マップ -->

### 全26ツール一覧

★ = 日常的に高頻度で使うツール

| セクション | ツールID | 説明 |
|-----------|---------|------|
| **Files** | `read` ★ | ファイルの内容を読む |
| | `write` ★ | ファイルを作成・上書き |
| | `edit` | ファイルを精密に編集 |
| **Runtime** | `exec` ★ | シェルコマンド実行 |
| | `process` | バックグラウンドプロセス管理 |
| **Web** | `web_search` ★ | Web検索（Brave Search API） |
| | `web_fetch` | WebページのコンテンツをMarkdown抽出 |
| **Memory** | `memory_search` | セマンティック検索 |
| | `memory_get` | メモリファイル読み込み |
| **Sessions** | `sessions_list` | セッション一覧 |
| | `sessions_history` | セッション履歴 |
| | `sessions_send` | セッションにメッセージ送信 |
| | `sessions_spawn` | サブエージェント生成 |
| | `sessions_yield` | サブエージェント結果待ち |
| | `subagents` | サブエージェント管理 |
| | `session_status` | セッション状態確認 |
| **UI** | `browser` | Webブラウザ制御 |
| | `canvas` | キャンバス制御 |
| **Messaging** | `message` ★ | メッセージ送信 |
| **Automation** | `cron` | タスクスケジュール |
| | `gateway` | Gateway制御 |
| **Nodes** | `nodes` | ノード＆デバイス管理 |
| **Agents** | `agents_list` | エージェント一覧 |
| **Media** | `image` | 画像認識・理解 |
| | `image_generate` | 画像生成 |
| | `tts` | テキスト読み上げ |

※ 上記に加え、`apply_patch`（OpenAI形式のパッチ適用）が存在するが、これは `exec` が許可されていれば自動的に利用可能になる特例ツールのため、独立カウントには含めていない。

### 代表的なツールをピックアップ

**Files（ファイル操作）— 3ツール ＋ apply_patch**

開発作業の基本中の基本。`read` でファイルを読み、`write` で書き、`edit` で精密編集する。`apply_patch` はOpenAI形式のパッチ適用に対応しており、`exec` が許可されていれば自動的に利用可能になる特例がある。

**Runtime（実行環境）— 2ツール**

`exec` はOpenClawの中でも最も強力なツールの一つ。任意のシェルコマンドを実行できる。`process` はバックグラウンドで実行中のプロセスを管理する。この2つがあるからこそ、エージェントは「会話するだけのAI」を超えた存在になれる。

**Web（Web操作）— 2ツール**

`web_search` はBrave Search APIでWeb検索を実行し、`web_fetch` はURLからコンテンツをMarkdown形式で抽出する。情報収集の要だ。

**Sessions（セッション管理）— 7ツール**

第5章で学んだセッション管理の実行インターフェース。特に `sessions_spawn` と `sessions_yield` はサブエージェント生成に不可欠で、複雑なタスクを並行処理する鍵になる。`session_status` は唯一 `minimal` プロファイルでも使えるツールだ。

**Messaging（メッセージング）— 1ツール**

前章で登場した `message` ツール。Discord、Telegram、WhatsApp——どのプラットフォームにも、この1つのツールで送信できる。チャンネルの「翻訳者」がプラットフォームごとの差異を吸収してくれるのだ。

**Media（メディア）— 3ツール**

`image` で画像を理解し、`image_generate` で画像を生成し、`tts` でテキストを音声に変換する。マルチモーダルなエージェント体験の入口である。

### ツール名エイリアスとツールグループ

OpenClawにはツール名の**エイリアス**がある。たとえば `bash` と入力しても `exec` として認識される。`apply-patch` は `apply_patch` のエイリアスだ。

また、ポリシー設定で便利な**ツールグループ**が用意されている。代表的なグループを挙げる：

- `group:fs` — ファイル操作ツール（read, write, edit, apply_patch）
- `group:runtime` — 実行ツール（exec, process）
- `group:web` — Web操作ツール（web_search, web_fetch）
- `group:sessions` — セッション管理ツール
- `group:messaging` — メッセージングツール
- `group:media` — メディアツール
- `group:plugins` — プラグイン提供ツール全体
- `group:openclaw` — OpenClawコアツール群（`includeInOpenClawGroup: true` のツール）

グループ名はツールポリシー（8.4節で詳述）で威力を発揮する。

> 💬 **こうじの実感：** 26個って聞くと多い気がするけど、実際によく使うのはread / write / exec / web_search / messageの5つくらい。残りは必要になった時に「あ、こんなのもあったな」って思い出す感じ。全部覚える必要はない。

---

## 8.3 プロファイル — 場面に応じたツールセットの切り替え

26個のツールすべてを常に使えるようにするのは、セキュリティ上望ましくない。グループチャットで `exec` が使えてしまったら怖いだろう？そこで登場するのが**プロファイル**だ。

### 4つのプロファイル

OpenClawには4つのプロファイルが用意されている：

| プロファイル | 利用可能ツール数 | 主な用途 |
|-------------|-----------------|---------|
| **minimal** | 1個 | 最小権限。`session_status` のみ |
| **coding** | 20個 | 開発作業。ファイル操作・実行・Web検索・サブエージェント等 |
| **messaging** | 5個 | メッセージ応答特化。セッション操作とmessageのみ |
| **full** | 全ツール | 制限なし |

<!-- IMAGE: ch08-profile-comparison.png - 4つのプロファイルで利用可能なツール数と代表ツールを比較した図 -->

### 各プロファイルの詳細

**minimal — 1個**
- `session_status` のみ
- 最も制限の厳しいプロファイル。「何もさせない」が基本方針

**coding — 20個**
- `read`, `write`, `edit`, `apply_patch`（ファイル操作）
- `exec`, `process`（実行環境）
- `web_search`, `web_fetch`（Web操作）
- `memory_search`, `memory_get`（メモリ）
- `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status`（セッション管理）
- `cron`（自動化）
- `image`, `image_generate`（メディア）
- 開発作業に必要な主要ツールが網羅されている

**messaging — 5個**
- `sessions_list`, `sessions_history`, `sessions_send`, `session_status`（セッション操作）
- `message`（メッセージ送信）
- チャット応答に必要最小限のツールセット

**full — 全ツール**
- `browser`, `canvas`, `gateway`, `nodes`, `agents_list`, `tts` など、他のプロファイルに含まれないツールもすべて利用可能

### プロファイルの使い分け

プロファイルはツールポリシーの「第一層」として機能する。たとえば：

- **DM（1対1チャット）** → `coding` プロファイル：開発支援に必要なツールをフル活用
- **グループチャット** → `messaging` プロファイル：メッセージ応答だけに制限
- **公開API経由** → `minimal` プロファイル：最小権限で安全に

場面に応じて武器を制限する発想は、セキュリティの基本原則「最小権限の原則（Principle of Least Privilege）」そのものである。

> 💬 **こうじの実感：** グループチャットでexecツール使えちゃったら怖いでしょ？messagingプロファイルはそういう事故を防ぐためにある。場面に応じて武器を制限する発想、セキュリティ的に超大事。

---

## 8.4 ツールポリシー — 「何を許可するか」の精密制御

プロファイルだけではまだ粒度が粗い。「codingプロファイルだけど `cron` は使わせたくない」「messagingプロファイルだけど `web_search` も使わせたい」——こうした細かな制御を実現するのが**ツールポリシー**だ。

### allow / deny グロブパターン

ツールポリシーは `allow`（許可）と `deny`（拒否）のグロブパターンリストで構成される：

```yaml
tools:
  sandbox:
    tools:
      allow: ["read", "write", "exec", "group:web"]
      deny: ["gateway"]
```

### ポリシー評価ロジック

評価は以下の順序で行われる：

1. `deny` リストにマッチ → **拒否**（denyは最優先）
2. `allow` リストが空 → **許可**（明示的な制限なし＝全許可）
3. `allow` リストにマッチ → **許可**
4. 特例：`apply_patch` は `exec` が許可されていれば自動的に許可
5. それ以外 → **拒否**

**denyが最優先**というのが重要なポイントだ。`allow: ["*"]` で全許可にしても、`deny: ["gateway"]` があればgatewayは使えない。

### 6つのポリシーレイヤー

ここからが核心。OpenClawでは、複数のポリシーが**重畳的に**適用される。すべてのレイヤーで許可されて初めて、そのツールが使える（AND評価）。

**まずは2つだけ覚えればOK：** 日常的に意識するのは「① プロファイルポリシー」と「② サンドボックスポリシー」の2つだけだ。残りの4レイヤーは上級者向けで、必要になった時に戻ってくればいい。

| レイヤー | 説明 | 設定箇所 |
|---------|------|---------|
| 1. プロファイルポリシー | `coding` / `messaging` / `minimal` / `full` の基本セット | プロファイル定義 |
| 2. サンドボックスポリシー | グローバルなallow / deny | `tools.sandbox.tools.allow` / `deny` |
| 3. エージェント固有 | エージェントごとの制限 | `agents.list[].tools.sandbox.tools` |
| 4. グループポリシー | チャンネル固有のツール制限 | チャンネル設定 |
| 5. オーナー限定ポリシー | 非オーナーから特定ツールを隠す | 自動適用 |
| 6. Gateway HTTP拒否リスト | HTTP API経由の呼び出し制限 | 自動適用 |

<!-- IMAGE: ch08-policy-layers.png - 6つのポリシーレイヤーが重畳的に評価されるフローチャート -->

### オーナー限定ツール

一部のツールはデフォルトでオーナーのみ利用可能になっている：

- `whatsapp_login` — WhatsAppログイン
- `cron` — タスクスケジュール
- `gateway` — Gateway制御
- `nodes` — ノード＆デバイス管理

非オーナーの送信者からは、これらのツールは完全に見えない状態になる。

### 危険なツール（Gateway HTTP拒否リスト）

Gateway HTTP API経由でデフォルト拒否されるツールもある：

- `sessions_spawn`, `sessions_send` — サブエージェント操作
- `cron` — スケジュールタスク
- `gateway` — Gateway制御
- `whatsapp_login` — WhatsAppログイン

これらはHTTP APIからの不正利用を防ぐための安全装置だ。

### alsoAllowによる追加許可

既存のポリシーに対して、追加で許可をマージする `alsoAllow` という仕組みもある：

```yaml
tools:
  sandbox:
    tools:
      alsoAllow: ["cron", "image_generate"]
```

これにより、基本ポリシーを変更せずに特定ツールだけ追加許可できる。なお、既存の `allow` リストが未設定（＝全許可状態）の場合、`alsoAllow` はその全許可に加えて指定ツールを明示的に許可リストへ追加する動作となる。

### 実践的な設定例

```yaml
# サンドボックスツールポリシー
tools:
  sandbox:
    tools:
      allow: ["read", "write", "exec", "web_search"]
      deny: ["gateway"]
      alsoAllow: ["cron"]

# エージェント固有のポリシー
agents:
  list:
    - name: research-agent
      tools:
        sandbox:
          tools:
            allow: ["group:fs", "group:web"]
            deny: []
```

> 💬 **こうじの実感：** 正直、ポリシーの重ね合わせは最初ややこしかった。でも「全部のレイヤーでOKじゃないと通らない」って覚えておけば大丈夫。サンドボックスでexec許可しても、プロファイルがmessagingならそもそも使えない。二重三重のチェック。

> 💡 **コラム：ツールグループで楽々ポリシー管理**
>
> ポリシー設定で個別ツール名を列挙するのは面倒だ。そこで活用したいのが**ツールグループ**。`group:fs`、`group:web` 等のグループ名をallow / denyに使えば、一発で複数ツールを制御できる。
>
> ```yaml
> tools:
>   sandbox:
>     tools:
>       allow: ["group:fs", "group:web"]
> ```
>
> これだけで「ファイル操作とWeb検索だけ許可」が実現する。`group:plugins` を使えばプラグイン提供ツールもまとめて制御可能だ。ツールが増えても、グループ単位で管理していれば設定が肥大化しない。

---

## 8.5 スキルとは何か — 汎用エージェントを専門家に変える仕組み

ここからはスキルの世界に入る。ツール（筋肉）の全体像を理解した今、スキル（技術書）の仕組みを深掘りしよう。

### スキル＝「特定分野の専門マニュアル」を丸ごとパッケージにしたもの

スキルを一言で言えば、「特定ドメインへのオンボーディングガイド」だ。汎用エージェントを、天気予報の専門家に、GitHub操作の専門家に、音声合成の専門家に変える仕組みである。技術的には、必要なファイルが1つのフォルダにまとまった**モジュール化された自己完結型パッケージ**として実装されている。

### スキルが提供する4つのもの

1. **専門ワークフロー** — 特定ドメインの多段階手順（例：天気予報の取得手順）
2. **ツール統合** — 特定のファイル形式やAPIとの連携手順（例：Notion APIの叩き方）
3. **ドメイン知識** — 企業固有の知識、スキーマ、ビジネスロジック
4. **バンドルリソース** — スクリプト、リファレンスドキュメント、テンプレート等のアセット

### ディレクトリ構造

スキルのファイル構成はシンプルだ：

```
skill-name/
├── SKILL.md          (必須) スキル定義＋手順書
│   ├── YAMLフロントマター (必須: name, description)
│   └── Markdown本文      (手順・ガイダンス)
└── バンドルリソース    (任意)
    ├── scripts/      実行可能コード（Python / Bash等）
    ├── references/   必要に応じて読み込むドキュメント
    └── assets/       出力に使用するファイル（テンプレート等）
```

最小構成は**フォルダ + SKILL.md のみ**。それだけでスキルとして機能する。

<!-- IMAGE: ch08-skill-structure.png - スキルのディレクトリ構造とSKILL.mdの構成要素を図解 -->

### SKILL.mdのフロントマター

SKILL.mdの先頭にはYAMLフロントマターを記述する。最低限必要なのは `name` と `description` の2つだ。

```yaml
---
name: weather
description: >
  Get current weather and forecasts via wttr.in or Open-Meteo.
  Use when: user asks about weather, temperature, or forecasts for any location.
  NOT for: historical weather data, severe weather alerts.
  No API key needed.
---
```

ここで**descriptionが極めて重要**だということを強調しておく。この説明文が、スキル発動の主要なトリガーメカニズムになるのだ（詳しくは8.8節で解説する）。

### Progressive Disclosure — 3段階ロード

スキルの設計で天才的なのが**Progressive Disclosure（段階的開示）**だ。約50個のスキルを一度にすべて読み込んだら、コンテキストウィンドウが溢れてしまう。そこでOpenClawは、スキルを3段階でロードする：

| 段階 | 何がロードされるか | タイミング | サイズ目安 |
|-----|-------------------|-----------|-----------|
| 1. メタデータ | name + description | 常にコンテキストに含まれる | 約100ワード |
| 2. SKILL.md本文 | Markdown手順書 | スキルがトリガーされた時 | 5,000ワード未満推奨 |
| 3. バンドルリソース | scripts/, references/, assets/ | エージェントが必要に応じて読み込み | サイズ制限なし |

人間が本を読むとき、まず目次（メタデータ）を見て、必要な章（SKILL.md本文）だけ読み、さらに詳しい情報が必要なら付録（バンドルリソース）を参照する——スキルの3段階ロードは、まさにこの自然な読書行動と同じ発想なのである。

> 💬 **こうじの実感：** スキルの3段階ロードは天才的な設計だと思う。50個のスキルのSKILL.mdを全部読み込んだらコンテキスト溢れる。最初はdescriptionだけ見て、必要な時だけ中身を読む。人間の「目次だけ見て必要な章だけ読む」と同じ発想。

---

## 8.6 組み込みスキル約50種 — カテゴリ別ガイドツアー

OpenClawにはバンドルスキルとして**約50個の組み込みスキル**が用意されている。これらは `/usr/lib/node_modules/openclaw/skills/` に格納されており、追加インストールなしで利用可能だ。10のカテゴリに分類して見ていこう。

<!-- IMAGE: ch08-builtin-skills-catalog.png - 10カテゴリに分類された組み込みスキルのカタログ風一覧図 -->

### 📱 メッセージング・コミュニケーション（10個）

| スキル名 | 説明 |
|---------|------|
| `discord` | Discord操作（messageツール経由） |
| `slack` | Slack操作（message / slackツール経由） |
| `bluebubbles` | iMessage送受信（BlueBubbles経由） |
| `imsg` | iMessage / SMS CLI（Messages.app経由） |
| `wacli` | WhatsAppメッセージ送信・履歴検索 |
| `himalaya` | IMAP / SMTPメール管理CLI |
| `gog` | Google Workspace CLI（Gmail, Calendar, Drive等） |
| `trello` | Trello REST API操作 |
| `notion` | Notion API操作 |
| `voice-call` | OpenClaw音声通話プラグイン |

### 🏠 スマートホーム・IoT（4個）

| スキル名 | 説明 |
|---------|------|
| `openhue` | Philips Hueライト・シーン制御 |
| `eightctl` | Eight Sleepポッド制御 |
| `sonoscli` | Sonosスピーカー制御 |
| `blucli` | BluOS CLI |

### 🎵 メディア・音声（8個）

| スキル名 | 説明 |
|---------|------|
| `sag` | ElevenLabs TTS |
| `sherpa-onnx-tts` | ローカルTTS（オフライン） |
| `openai-whisper` | ローカル音声認識（Whisper CLI） |
| `openai-whisper-api` | OpenAI音声認識API |
| `spotify-player` | Spotify再生・検索 |
| `songsee` | オーディオスペクトログラム生成 |
| `video-frames` | 動画からフレーム抽出 |
| `gifgrep` | GIF検索・ダウンロード |

### 📝 ノート・タスク管理（5個）

| スキル名 | 説明 |
|---------|------|
| `apple-notes` | Apple Notes管理（memo CLI） |
| `apple-reminders` | Apple Reminders管理 |
| `bear-notes` | Bear Notes管理（grizzly CLI） |
| `things-mac` | Things 3タスク管理 |
| `obsidian` | Obsidianボールト操作 |

### 🛠️ 開発・CI（5個）

| スキル名 | 説明 |
|---------|------|
| `github` | GitHub操作（gh CLI） |
| `gh-issues` | GitHubイシュー取得・PR自動化 |
| `coding-agent` | コーディングタスクを外部エージェントに委任 |
| `tmux` | tmuxセッション制御 |
| `session-logs` | セッションログ分析 |

### 📄 ドキュメント処理（2個）

| スキル名 | 説明 |
|---------|------|
| `nano-pdf` | PDF編集（自然言語指示） |
| `summarize` | URL・ポッドキャスト・ファイルの要約 |

### 🔍 検索・情報（3個）

| スキル名 | 説明 |
|---------|------|
| `weather` | 天気予報（wttr.in / Open-Meteo） |
| `blogwatcher` | ブログ・RSS / Atomフィード監視 |
| `goplaces` | Google Places API検索 |

### 📷 カメラ・画面（3個）

| スキル名 | 説明 |
|---------|------|
| `camsnap` | RTSP / ONVIFカメラからフレーム / クリップ取得 |
| `peekaboo` | macOS UI操作・キャプチャ |
| `canvas` | キャンバス制御 |

### 🔐 セキュリティ・認証（1個）

| スキル名 | 説明 |
|---------|------|
| `1password` | 1Password CLI操作 |

### 🌐 外部サービス（5個）

| スキル名 | 説明 |
|---------|------|
| `xurl` | X（Twitter）API操作 |
| `ordercli` | Foodora注文履歴確認 |
| `mcporter` | MCPサーバー / ツール呼び出し |
| `gemini` | Gemini CLI Q&A |
| `oracle` | Oracle CLI操作 |

### ⚙️ OpenClaw管理（4個）

| スキル名 | 説明 |
|---------|------|
| `skill-creator` | スキル作成・編集・監査 |
| `clawhub` | ClawHub CLIでスキル検索・インストール・公開 |
| `healthcheck` | ホストセキュリティ強化 |
| `node-connect` | ノード接続診断 |

### その他（1個）

| スキル名 | 説明 |
|---------|------|
| `model-usage` | モデル使用量・コスト確認 |

### 中身を覗いてみよう

「スキルって何が書いてあるの？」と思うだろう。`weather` スキルを例に中身を見てみよう。

実は `weather` スキルの中身は「`exec` ツールで `curl wttr.in` を叩く手順が書いてあるだけ」なのだ。しかし、それが強力である。エージェントは天気APIの叩き方なんて知らないが、スキルを読めば一発でできる。これがスキルの本質——**既存のツールに専門知識を与える**ことなのである。

> 💬 **こうじの実感：** 約50個もあるのに、実際に名前と中身を見るまで知らないスキルだらけだった。「こんなのもあるの？」って発見が楽しい。自分に必要なやつだけ覚えておいて、あとは必要になった時に探せばいい。

---

## 8.7 スキルの発見と読み込み — 7つのソースと優先順位

スキルは複数の場所から読み込まれる。そしてその優先順位を理解することが、カスタマイズの鍵になる。

### 7つの読み込みソース（優先順位順）

スキルは以下の場所から読み込まれる。**後のソースが前のソースを上書きする**（Map.set()方式）：

| 優先度 | ソース名 | パス | 説明 |
|-------|---------|------|------|
| 低 | extraDirs | `config.skills.load.extraDirs` で指定 | 追加ディレクトリ |
| ↑ | プラグインスキル | プラグインの `skills` フィールド | プラグイン提供 |
| ↑ | バンドルスキル | `/usr/lib/node_modules/openclaw/skills/` | OpenClaw同梱（約50個） |
| ↑ | マネージドスキル | `~/.openclaw/skills/` | OpenClaw管理 |
| ↑ | パーソナルAgentsスキル | `~/.agents/skills/` | 個人用 |
| ↑ | プロジェクトAgentsスキル | `<workspace>/.agents/skills/` | プロジェクト用 |
| 高 | ワークスペーススキル | `<workspace>/skills/` | 最優先 |

<!-- IMAGE: ch08-skill-loading-pipeline.png - 7つのソースからスキルが読み込まれ、フィルタリング・プロンプト注入される流れ -->

### 同名スキルの上書き

最も重要なポイントは、**ワークスペーススキルが最優先**ということだ。バンドルの `weather` スキルが気に入らなければ、自分の `<workspace>/skills/weather/SKILL.md` に同名で配置すれば上書きできる。カスタマイズの自由度が高い設計である。

### 読み込みフロー

スキルの読み込みは以下のステップで行われる：

1. 各ソースディレクトリの子ディレクトリをスキャン
2. 各子ディレクトリに `SKILL.md` があるか確認
3. `SKILL.md` のファイルサイズチェック（デフォルト最大256KB）
4. `loadSkillsFromDir()` でパース（`@mariozechner/pi-coding-agent` ライブラリ使用）
5. パスがルート外に脱出していないかセキュリティチェック
6. フロントマター解析 → メタデータ・呼び出しポリシー解決
7. `shouldIncludeSkill()` でフィルタリング（OS、依存バイナリ、環境変数、設定等）

### 上限設定

スキルが無制限に増えるとパフォーマンスに影響するため、各種上限が設けられている：

| 設定 | デフォルト値 |
|-----|------------|
| `maxCandidatesPerRoot` | 300 |
| `maxSkillsLoadedPerSource` | 200 |
| `maxSkillsInPrompt` | 150 |
| `maxSkillsPromptChars` | 30,000文字 |
| `maxSkillFileBytes` | 256,000バイト（256KB） |

### プロンプトへの注入

読み込まれたスキルは、システムプロンプトの `<available_skills>` XMLブロックとして注入される：

```xml
<available_skills>
  <skill>
    <name>weather</name>
    <description>Get current weather and forecasts via wttr.in...</description>
    <location>/usr/lib/node_modules/openclaw/skills/weather/SKILL.md</location>
  </skill>
  <skill>
    <name>discord</name>
    <description>Discord ops via the message tool...</description>
    <location>/usr/lib/node_modules/openclaw/skills/discord/SKILL.md</location>
  </skill>
  ...
</available_skills>
```

ホームディレクトリのパスは `~` に短縮され、1スキルあたり5-6トークンが節約される。

> 💬 **こうじの実感：** ワークスペーススキルが最優先ってのがポイント。バンドルのweatherスキルが気に入らなければ、自分のworkspace/skills/weatherに同名で置けば上書きできる。カスタマイズの自由度が高い。

> 💡 **コラム：スキルの文字数予算とcompact format**
>
> スキルが増えすぎるとシステムプロンプトが肥大化する。OpenClawはこの問題に対して、30,000文字の予算を設けている。
>
> 通常は各スキルの `name` + `description` + `location` がフル表示されるが、文字数予算を超過すると**compact format**に自動切り替えされる。compact formatでは `description` が省略され、`name` と `location` のみになる。
>
> それでも超過する場合は**バイナリサーチ**で収まる数まで削減される。「100個は多すぎる？50個なら？75個なら？」と半分ずつ試して、予算内に収まるスキル数を効率的に決定するのだ。
>
> 開発者が気にしなくても、OpenClawが自動的にコンテキストを最適化してくれる——第6章のコンテキストエンジンと同じ思想がここにも生きている。

---

## 8.8 スキルの発動メカニズム — descriptionが鍵

スキルがシステムプロンプトに注入されることはわかった。では、**いつ、どのスキルが発動するのか？** その鍵を握るのが `description` フィールドだ。

### 発動の流れ

1. ユーザーが質問する（「今日の天気は？」）
2. エージェントは `<available_skills>` ブロック内のdescriptionを参照する
3. 質問にマッチするスキルを見つける（`weather` スキルの description に "weather, temperature, forecasts" とある）
4. エージェントが `read` ツールで SKILL.md を読み込む
5. SKILL.md の手順に従い、既存ツール（`exec` 等）を呼び出す
6. 結果をユーザーに返す

<!-- IMAGE: ch08-skill-trigger-flow.png - ユーザー質問からスキル発動、ツール実行、結果返却までのシーケンス図 -->

### スラッシュコマンドによる明示的呼び出し

自動発動だけでなく、ユーザーが `/スキル名` のスラッシュコマンドで明示的にスキルを呼び出すこともできる。

コマンド名のルール：
- 英小文字・数字・アンダースコアのみ
- 最大32文字
- 重複する場合は `_2`, `_3` 等のサフィックスが自動追加

### 発動制御のフロントマター

スキルの発動方法を細かく制御するフロントマターもある：

```yaml
---
name: my-skill
description: "..."
user-invocable: true              # /my-skill コマンドで呼べるか（デフォルト: true）
disable-model-invocation: false   # モデルの自動発動を禁止するか（デフォルト: false）
command-dispatch: tool            # コマンド実行方式（toolでツール直接ディスパッチ）
command-tool: exec                # ディスパッチ先ツール名
allowed-tools: ["message", "exec"]  # このスキルが使用するツールの宣言
---
```

- `user-invocable: false` にすると、スラッシュコマンドとしては使えなくなるが、モデルの自動発動は可能
- `disable-model-invocation: true` にすると、モデルが勝手にスキルを発動しなくなり、ユーザーの明示的コマンドでのみ動作する
- `command-dispatch: tool` を使えば、SKILL.mdを読まずにツールに直接ディスパッチできる

> 💬 **こうじの実感：** descriptionの書き方がスキルの命運を握ってる。曖昧に書くと発動しないし、広すぎると関係ないタスクでも発動しちゃう。「いつ使うか、何ができるか」を明確に書くのがコツ。weatherスキルの description に "NOT for: historical weather data" って除外条件まで書いてあるの、よくできてるなと思う。

---

## 8.9 カスタムスキルの作成 — 自分だけの専門家を作る

ここまでで組み込みスキルの仕組みを理解した。次は**自分だけのスキルを作ってみよう**。驚くほど簡単だ。

### スキルの配置場所

カスタムスキルは以下のいずれかに配置する：

| 場所 | パス | 用途 |
|-----|------|------|
| ワークスペース（最優先） | `<workspace>/skills/<skill-name>/` | プロジェクト固有 |
| パーソナル | `~/.agents/skills/<skill-name>/` | 全プロジェクト共通 |
| プロジェクトAgents | `<workspace>/.agents/skills/<skill-name>/` | チーム共有 |

### 最小構成

スキル作成に必要な最小構成は、フォルダとSKILL.mdだけだ：

```
my-skill/
└── SKILL.md
```

### ステップバイステップ：翻訳スキルを作ってみる

実際にカスタムスキルを1つ作ってみよう。「英語テキストを日本語に翻訳するスキル」だ。

**Step 1: フォルダ作成**

```bash
mkdir -p ~/.openclaw/workspace/skills/translate-en-ja
```

**Step 2: SKILL.mdを書く**

```yaml
---
name: translate-en-ja
description: >
  Translate English text to Japanese.
  Use when: user provides English text and asks for Japanese translation,
  or when user says "翻訳して", "日本語にして", "translate to Japanese".
  NOT for: Japanese to English, or other language pairs.
---

# English to Japanese Translation

## Instructions

1. Read the provided English text carefully.
2. Translate it to natural Japanese, maintaining the original tone and nuance.
3. Use appropriate register (formal/casual) based on the source text.
4. For technical terms, provide the Japanese translation with the original English in parentheses on first occurrence.

## Output Format

- Present the translation directly without preamble.
- If the text is long, translate paragraph by paragraph.
- Preserve formatting (headings, lists, code blocks) from the original.
```

**Step 3: テスト**

OpenClawに「Translate this to Japanese: "The quick brown fox jumps over the lazy dog."」と話しかけてみる。descriptionの "translate to Japanese" にマッチし、SKILL.mdの手順に従って翻訳が実行されるはずだ。

<!-- IMAGE: ch08-custom-skill-creation.png - カスタムスキル作成のステップバイステップ図（フォルダ作成→SKILL.md記述→テスト→配布） -->

### 命名規則

- **小文字・数字・ハイフンのみ**
- **64文字以内**
- フォルダ名 = スキル名
- 短い動詞から始まるフレーズ推奨
- ツール名でネームスペース化も可能（例：`gh-address-comments`）

### SKILL.mdのベストプラクティス

- **500行以下**を推奨（コンテキストウィンドウ節約）
- 命令形・不定詞形で記述する
- エージェントが既に知っている情報は含めない
- 詳細情報は `references/` サブディレクトリに分離する
- トリガー条件は **description** に書く（本文ではなく）

### 高度なフロントマター

APIキーが必要なスキルや、特定のバイナリに依存するスキルには、より高度なフロントマターを使う：

```yaml
---
name: my-api-skill
description: "..."
metadata:
  openclaw:
    emoji: "🔧"
    primaryEnv: MY_API_KEY
    requires:
      bins: ["curl"]
      env: ["MY_API_KEY"]
    os: ["linux", "darwin"]
    install:
      - id: brew
        kind: brew
        formula: my-tool
        bins: ["my-tool"]
        label: "Install my-tool (brew)"
      - id: node
        kind: node
        package: my-tool
        bins: ["my-tool"]
        label: "Install my-tool (npm)"
---
```

`requires` で依存条件を宣言しておけば、条件を満たさない環境ではスキルが自動的にフィルタリングされる。たとえば `os: ["darwin"]` と書けばmacOSでのみ有効になる。

### skill-creatorスキルによる作成支援

スキル作成をさらに楽にするために、`skill-creator` という組み込みスキルがある。`scripts/init_skill.py` でスキルの雛形を自動生成できる：

```bash
# 初期化
scripts/init_skill.py my-skill --path skills/public --resources scripts,references

# パッケージング（.skillファイル生成、バリデーション付き）
scripts/package_skill.py path/to/skill-folder
```

### ClawHubによる配布

作成したスキルは `clawhub` CLIで検索・インストール・公開できる。コミュニティとスキルを共有するエコシステムが整備されているのだ。

> 💬 **こうじの実感：** 実際にtranslate-articleとresearch-delegateっていう自作スキルを使ってる。最初は「スキル作るとか難しそう」と思ったけど、SKILL.mdにMarkdownで手順書くだけ。プログラミング不要。10分で作れる。

---

## 8.10 スキル関連の設定 — config.yamlでの細かな制御

カスタムスキルの作り方がわかったところで、`config.yaml` でのスキル関連設定を整理しよう。

### 全体設定

```yaml
skills:
  # バンドルスキルの許可リスト（未指定=全許可）
  allowBundled: ["weather", "discord", "github"]

  # 個別スキル設定
  entries:
    weather:
      enabled: true
      env:
        WEATHER_API_KEY: "your-api-key"
    discord:
      enabled: false  # 無効化

  # 読み込み設定
  load:
    extraDirs: ["/path/to/custom/skills"]

  # スキル数制限
  limits:
    maxCandidatesPerRoot: 300
    maxSkillsLoadedPerSource: 200
    maxSkillsInPrompt: 150
    maxSkillsPromptChars: 30000
    maxSkillFileBytes: 256000

  # インストール設定
  install:
    preferBrew: true
    nodeManager: "npm"  # npm, pnpm, yarn, bun
```

### 主要な設定項目

**`skills.allowBundled`** — バンドルスキルの許可リスト。指定すると、リストに含まれるスキルのみが有効になる。未指定なら全バンドルスキルが有効。

**`skills.entries`** — 個別スキルの有効/無効、環境変数、APIキーを設定する。

**`skills.load.extraDirs`** — 追加のスキル読み込みディレクトリを指定する。

**`skills.limits`** — 各種上限値を調整する。通常はデフォルトで十分だが、スキルが大量にある場合に調整が必要。

### 環境変数とAPIキーの注入

APIキーが必要なスキル（ElevenLabs TTS等）では、`entries` の `apiKey` フィールドを使う：

```yaml
skills:
  entries:
    sag:
      apiKey: "your-elevenlabs-api-key"
```

`apiKey` は、スキルの `primaryEnv` で指定された環境変数名に自動的にマッピングされる。たとえば `sag` スキルの `primaryEnv` が `ELEVENLABS_API_KEY` なら、上の設定で `ELEVENLABS_API_KEY=your-elevenlabs-api-key` がプロセス環境変数として注入される。

また、`env` フィールドで任意の環境変数を直接注入することもできる：

```yaml
skills:
  entries:
    my-skill:
      env:
        CUSTOM_API_URL: "https://api.example.com"
        CUSTOM_API_KEY: "xxx"
```

**セキュリティ上の注意：** スキル環境変数はACPハーネス起動時にストリップされる。これにより、サブプロセスへのAPIキー漏洩が防止される。

### 環境変数によるバンドルスキルディレクトリの上書き

`OPENCLAW_BUNDLED_SKILLS_DIR` 環境変数で、バンドルスキルの参照先ディレクトリを上書きすることもできる。開発やテスト時に便利だ。

> 💬 **こうじの実感：** APIキーが必要なスキル（ElevenLabs TTSとか）は、config.yamlのentries.スキル名.apiKeyに設定するだけ。primaryEnvで環境変数名に自動マッピングされるから、SKILL.md側でos.getenv()とかしなくていい。楽。

---

## 8.11 プラグインによるツール拡張 — スキルとの違い

最後に、スキルとよく混同される**プラグイン**との違いを明確にしておこう。

### プラグイン＝実際にツールを追加できる仕組み

スキルは「既存のツールの使い方を教える」ものだった。一方、プラグインは `api.registerTool()` を通じて**実際に新しいツールを追加できる**。これが根本的な違いだ。

| 比較軸 | スキル | プラグイン |
|--------|--------|-----------|
| 本質 | プロンプト注入（知識） | ツール定義の追加（機能） |
| ツール追加 | できない | できる（`api.registerTool()`） |
| 実装方法 | Markdownで手順を書く | TypeScript / JavaScript でコードを書く |
| 既存ツールとの関係 | 使い方を教える | 新しいツールを生み出す |
| 例 | weatherスキル → execで天気APIを叩く手順 | Firecrawlプラグイン → firecrawlツールを新規追加 |

### 具体例で理解する

**スキルの場合（weather）：**
1. weatherスキルのSKILL.mdに「`exec` ツールで `curl wttr.in` を叩け」と書いてある
2. エージェントは既存の `exec` ツールを使って天気情報を取得する
3. 新しいツールは追加されていない

**プラグインの場合（Firecrawl）：**
1. Firecrawlプラグインが `api.registerTool()` で `firecrawl` ツールを登録する
2. エージェントは新しい `firecrawl` ツールを直接呼び出せるようになる
3. 26個のコアツールに加えて、新しいツールが追加されている

### ツール登録の全体フロー

ツールがどのように登録されるか、全体の流れを見てみよう：

```
1. CORE_TOOL_DEFINITIONS（26個のコアツール定義）
      ↓
2. createOpenClawTools()（ランタイム用ツールオブジェクト生成）
      ↓
3. プラグイン api.registerTool()（追加ツール登録）
      ↓
4. applyOwnerOnlyToolPolicy()（オーナー限定ツールのフィルタリング）
      ↓
5. ツールポリシーパイプライン（6レイヤーの最終判定）
      ↓
6. エージェントが利用可能なツール一覧が決定
```

<!-- IMAGE: ch08-tool-registration-flow.png - ツール登録の全体フロー図（コア定義→ランタイム生成→プラグイン追加→ポリシー適用） -->

### プラグインツールのポリシー制御

プラグインが追加したツールも、通常のツールポリシーで制御できる：

- `group:plugins` — プラグイン提供ツールをまとめて制御
- `group:<pluginId>` — 特定プラグインのツールだけを制御

```yaml
tools:
  sandbox:
    tools:
      allow: ["group:openclaw", "group:plugins"]  # コアツール＋プラグインツール全許可
      deny: ["group:firecrawl"]                   # Firecrawlだけ拒否
```

### プラグインスキル

面白いことに、プラグインはツールだけでなく**独自のスキルも提供できる**。プラグインの `skills` フィールドでスキルディレクトリを指定すると、そのスキルもスキル読み込みパイプラインに組み込まれる。第7章で学んだチャンネルプラグインが独自のスキルを提供するケースもこのパターンだ。

> 💬 **こうじの実感：** スキルとプラグインの違いは最初混乱した。スキルは「使い方マニュアル」で、プラグインは「新しい道具そのもの」。weatherスキルはexecで外部CLIを叩く手順を教えるだけだけど、Firecrawlプラグインは全く新しいfirecrawlツールを追加する。この区別がわかると、OpenClawの拡張モデルがスッキリ見える。

---

## 8.12 まとめと次章予告

### 重要ポイントの振り返り

この章では、OpenClawの「行動力」の全貌を解き明かした。5つの重要ポイントを振り返ろう。

**1. ツール＝アクション（26個コア）、スキル＝知識注入（約50個組み込み）**

ツールはエージェントが実行できる関数（筋肉）。スキルはエージェントに手順知識を与えるプロンプト注入（技術書）。この根本的な違いを理解することが、OpenClawの設計思想を読み解く鍵だ。

**2. プロファイルとポリシーの多層セキュリティ**

4つのプロファイル（minimal / coding / messaging / full）と6つのポリシーレイヤーが重畳的に評価される。すべてのレイヤーで許可されて初めてツールが使える。最小権限の原則が徹底されている。

**3. スキルの3段階ロード（Progressive Disclosure）**

メタデータ→SKILL.md本文→バンドルリソースの3段階で段階的にロードされる。コンテキストウィンドウを効率的に使う賢い設計だ。

**4. カスタムスキルはSKILL.md一つで作成可能**

フォルダを作り、SKILL.mdにMarkdownで手順を書くだけ。プログラミング不要。10分で自分だけの専門家エージェントを作れる。

**5. プラグインは新ツールを追加、スキルは既存ツールの使い方を教える**

スキルはプロンプト注入のみでツール自体は追加しない。プラグインは `api.registerTool()` で実際にツールを追加する。この区別がOpenClawの拡張モデルの核心だ。

### 次章予告

ツールとスキルで「手足」と「技術書」を手に入れたOpenClaw。しかし、これまで見てきた動作はすべて「人間が話しかけたら動く」という受動的なものだった。

もし、毎朝8時に自動で天気予報を送ってくれたら？ サーバーの異常を検知して自動でアラートを飛ばしてくれたら？ 外部サービスからのWebhookを受けて即座に対応してくれたら？

**第9章「自動化（Cron / Heartbeat / Webhook）」** では、OpenClawが人間の指示なしに自律的に動く仕組みを解剖する。Cronジョブによるスケジュール実行、Heartbeatによる定期チェック、Webhookによる外部トリガー——エージェントが「待っている」状態から「自分で動く」状態に進化する、その仕組みのすべてが次章で明かされる。

受動のAIから、能動のAIへ。次章が、その転換点だ。
