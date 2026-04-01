# 第9章：自動化（Cron / Heartbeat / Webhook）

> _「寝てる間にメール整理しといて」——人間にとっては無茶振りでも、OpenClawにとっては日常業務だ。Heartbeatで定期的に目を覚まし、Cronで決まった時間にタスクをこなし、Webhookで外部イベントに即応する。エージェントは24時間365日、あなたの代わりに働き続ける。ただし、電気代は自己負担だ。_

---

前章でエージェントに「手足」（ツール）と「技術書」（スキル）を与えた。26個のコアツールと約50個の組み込みスキルにより、エージェントは「行動するAI」へと変貌を遂げた。しかし、どれだけ高性能な手足があっても、あなたが「やって」と言わなければ動かない。これではただの便利な部下であり、自律的なエージェントとは呼べない。

この章では、エージェントに**自律性**を与える。人間が指示しなくても動く仕組み——OpenClawの自動化アーキテクチャを解き明かしていこう。

## この章を読み終える頃にわかること

- Heartbeatの仕組みと `HEARTBEAT.md` による制御方法
- Cronジョブの3種類のスケジュール（at / every / cron式）と使い分け
- Webhookによる外部イベント連携（Gmail等）の設定手順
- 配信キューの裏側——エラー時のリトライと永久エラーの扱い
- 3つの自動化手段の使い分け判断基準

## 3つの自動化手段

OpenClawには3つの自動化手段がある。それぞれ役割が異なる。

- **Heartbeat（心臓）** — 定期巡回。30分おきに「何かやることある？」と自分で確認する
- **Cron（時計）** — 時間指定実行。「毎朝9時にニュースをまとめて」のようなピンポイント実行
- **Webhook（耳）** — 外部イベント駆動。Gmailに新着メールが届いたら即座に反応する

第8章で `cron` ツールがオーナー限定ツールとして登場したのを覚えているだろうか。あのツールの裏で何が動いているのか——それがこの章のテーマだ。

<!-- IMAGE: ch09-automation-overview.png - 3つの自動化手段（Heartbeat/Cron/Webhook）の全体像。中央にGatewayプロセス、そこから3方向に伸びる概念図 -->

---

## 9.1 Heartbeat — エージェントの「心臓」

Heartbeatから始めるのには理由がある。設定が最も簡単で、効果を実感しやすい。`config.yaml` に数行追加するだけで「勝手に動いてる！」という体験ができる。しかもCronの土台でもあるので、ここを理解すれば後の話がスムーズに入ってくる。

### 9.1.1 Heartbeatとは何か

Heartbeatは、**定期的なシステムイベント**をメインセッション（デフォルト）または隔離セッションに送信し、エージェントに「生存確認＋タスク実行」を促す仕組みだ。Gatewayプロセス内のスケジューラーが管理しており、デフォルトでは **30分**（`agents.defaults.heartbeat.every`）おきに発火する。

比喩で言えば「30分おきに起きて、やることリストを確認して、なければ二度寝する」——これがHeartbeatの動きだ。シンプルだが、これだけでエージェントが自律的に動く土台になる。

なお、**エージェントが他のリクエストを処理中の場合、Heartbeatはスキップされる**（requests-in-flightチェック）。二重に起こして混乱させない安全設計だ。「Heartbeatが発火しない」と思ったら、まずこのチェックを疑うとよい。

### 9.1.2 HEARTBEAT.md — やることリストの正体

Heartbeatが発火すると、エージェントはワークスペースの `HEARTBEAT.md` を読む。これがHeartbeatの「指示書」であり、エージェントの行動を制御するファイルだ。

実際の `HEARTBEAT.md` はこんな感じで書く：

```markdown
# HEARTBEAT.md
- [ ] memory/YYYY-MM-DD.md を確認して、未完了タスクがあれば対応
- [ ] Discordの未読メッセージをチェック
- [ ] 重要な更新があれば晃一さんにサマリーを送信
```

ここで覚えておきたいのが、**実質的に空**の判定ロジックだ。以下の条件をすべて満たす場合、APIコール自体がスキップされる（コスト節約のため）：

| `HEARTBEAT.md` の状態 | 動作 |
|----------------------|------|
| ファイルが存在しない | **スキップしない**（LLMに判断を委ねる） |
| 空ファイル / 空行のみ | スキップ |
| 見出し行のみ（`# ヘッダー`、`## セクション` 等） | スキップ |
| 空のリストアイテムのみ（`- [ ]`（テキストなし）等） | スキップ |
| テキスト付きリストアイテムあり（`- [ ] タスク内容`） | 通常実行 |
| その他の実質的な指示あり | 通常実行 |

> ⚠️ **ここに注意**: ファイルが「空」ならスキップされるが、ファイル自体が「存在しない」場合はスキップされない。明示的に無効化したいなら、コメント行のみのファイルを作ること。

### 9.1.3 HEARTBEAT_OK — 「特にやることなし」の作法

エージェントが `HEARTBEAT.md` を読んで「やることなし」と判断した場合、`HEARTBEAT_OK` というトークンを返す。このトークンが返ると、OpenClawは以下の処理を行う：

- **トランスクリプト（会話履歴）を元のサイズに切り戻し**——「何もなかった」やり取りでコンテキストが汚染されるのを防ぐ
- **配信をスキップ**——ユーザーに「特にありません」とは通知しない

ここには細かい判定ロジックがある。`HEARTBEAT_OK` に短い付記（300文字以下）が付いている場合もスキップ扱いになる。一方、`HEARTBEAT_OK` + 長いテキストが返ってきた場合は実質的なレポートとみなして配信される。

**関連メカニズム:** 似た仕組みに `NO_REPLY`（`SILENT_REPLY_TOKEN`）というトークンもある。こちらはHeartbeatとは独立した「静かな返答」メカニズムで、テキスト全体がこのトークンのみの場合にメッセージ配信をスキップするものだ。

### 9.1.4 Heartbeatの設定オプション

`config.yaml` でHeartbeatの振る舞いを細かく制御できる：

```yaml
# config.yaml の設定例
agents:
  defaults:
    heartbeat:
      every: "30m"          # 間隔（デフォルト: 30分）
      target: "discord"     # 結果の配信先
      activeHours:
        start: "09:00"      # アクティブ時間の開始
        end: "23:00"        # アクティブ時間の終了
        timezone: "Asia/Tokyo"
```

各設定の意味を見ていこう。

**target（配信先）**

Heartbeat実行結果をどこに届けるかを指定する。

- `"none"`（デフォルト） — 実行はするが結果をチャットに送らない
- `"last"` — 最後にメッセージを受信したチャンネルに送る
- チャンネルID（例: `"discord"`） — 特定チャンネルに固定

> ⚠️ **ここに注意**: `target: "none"` はHeartbeat自体を無効にするのではなく、「実行するが結果を送らない」という意味だ。Heartbeat自体を無効にするには `every` を `0` にするか、負の値にする。

**activeHours（静粛時間）**

深夜にHeartbeatが発火してDiscordに通知が飛ぶのを防ぐための設定だ。`start` と `end` で時間範囲を指定し、範囲外の時間はスキップされる。タイムゾーンは `"user"`、`"local"`、またはIANAタイムゾーン（`"Asia/Tokyo"` 等）で指定可能だ。

ちなみに `start === end` にすると常にスキップ（24時間非アクティブ）になり、実質的なHeartbeat無効化として使える。

**isolatedSession**

`true` にするとメインセッションを汚さない隔離セッションでHeartbeatが実行される。メインの会話履歴にHeartbeatの痕跡を残したくない場合に便利だ。

<!-- IMAGE: ch09-heartbeat-flow.png - Heartbeatの実行フロー図。タイマー発火→前提チェック→HEARTBEAT.md読み込み→LLM実行→HEARTBEAT_OK判定→配信/スキップ -->

> 💬 **こうじの実感：** 最初は「設定項目多すぎない？」って思ったけど、ぶっちゃけ `every: "30m"` と `target: "discord"` だけ設定すればすぐ動く。activeHoursは深夜の通知がウザくなってから設定すればOK。isolatedSessionは上級者向けだから、最初は気にしなくていいよ。

### 9.1.5 エージェント別Heartbeat（上級）

マルチエージェント構成を使っている場合、エージェントごとに個別のHeartbeat設定が可能だ。`agents.list[].heartbeat.*` でデフォルト設定をオーバーライドできる。

重要なのは、明示的にHeartbeat設定を持つエージェントのみが発火するという点だ。たとえばエージェントAにだけ `heartbeat` フィールドを設定すれば、エージェントBのHeartbeatは動かない。マルチエージェント構成での無駄なAPIコールを防ぐ仕組みだ。

---

## 9.2 Cronジョブ — 時間指定のタスク実行

Heartbeatが「定期巡回のパトロール」だとすれば、Cronは「予約した仕事を時間通りにやる」仕組みだ。Unix cronの概念をAIエージェント向けに拡張したもので、3種類のスケジュール × 2種類のペイロード × 4種類のセッションターゲットという柔軟な組み合わせが特徴だ。

### 9.2.1 CronとHeartbeatの関係

CronジョブはHeartbeatと同じく**Gatewayプロセス内**で管理・実行される。そして実は、CronジョブがHeartbeatを「起こす」ことがある——この2つは密接に連携している。

CronジョブがHeartbeatと絡む経路は2つある：

- **systemEvent + main** → メインセッションにシステムイベントとして注入 → Heartbeat経由でエージェントが処理
- **agentTurn + isolated** → 隔離セッションでエージェントが直接実行 → 結果を配信

結論を先に言うと、**systemEventは「メインセッションの会話履歴を使いたいとき」に選ぶ**。それ以外はagentTurnでOKだ。

具体的なシナリオで見てみよう。**「毎日18時に日報をまとめる」をsystemEventで設定した場合**：

1. 18:00になるとCronジョブが発火し、「日報をまとめてDiscordに投稿して」というテキストをメインセッションのイベントキューに注入する
2. `wakeMode: "now"`（デフォルト）なので、即座にHeartbeatがトリガーされる
3. Heartbeatがイベントキューを確認し、Cronイベントを検出。このとき **HEARTBEAT.mdの空チェックはバイパスされる**（Cronイベントがあれば必ず処理する）
4. エージェントがメインセッションのコンテキスト（過去の会話、記憶）を参照しながら日報を作成
5. 結果が `heartbeat.target` で指定されたチャンネルに配信される

一方、**同じタスクをagentTurnで設定した場合**：

1. 18:00にCronジョブが発火し、隔離セッションでエージェントターンを直接開始する
2. エージェントはフレッシュなセッションで日報を作成（メインセッションのコンテキストは参照しない）
3. 結果は `--announce --channel discord` の設定に従って直接配信される

<!-- IMAGE: ch09-cron-heartbeat-relation.png - CronとHeartbeatの2つの経路を示すフロー図。systemEvent経路とagentTurn経路の分岐 -->

> 💬 **こうじの実感：** 最初「CronとHeartbeatって何が違うの？」って混乱したんだけど、Heartbeatは「定期巡回のパトロール」、Cronは「予約した仕事を時間通りにやる」って考えるとスッキリした。しかもCronの結果をHeartbeat経由で届けることもあるから、パトロール中に「あ、さっきの予約仕事の結果出てた」って拾ってくる感じ。

### 9.2.2 3種類のスケジュール

Cronジョブには3種類のスケジュールがあり、**1つだけ**指定が必須だ。

| 種類 | 用途 | 例 |
|------|------|-----|
| `at` | ワンショット（1回きり） | `--at 2026-04-02T09:00:00 --tz Asia/Tokyo` / `--at 20m` |
| `every` | インターバル（定期） | `--every 1h` / `--every 30m` |
| `cron` | cron式（柔軟な繰り返し） | `--cron "0 9 * * 1-5"` = 平日9時 |

**`at`（ワンショット）** は1回きりの実行に使う。ISO日時または相対的な持続時間（`20m`、`1h` 等）で指定できる。`--tz` でタイムゾーン指定も可能だ。

**`every`（インターバル）** はシンプルな定期実行。`10m`、`1h`、`1d` のような持続時間形式で間隔を指定する。

**`cron`（cron式）** は最も柔軟なスケジュール。5フィールド（分 時 日 月 曜日）または6フィールド（秒を含む）のcron式で細かく制御できる。

実際のコマンド例を見てみよう：

```bash
# 20分後にワンショット実行
openclaw cron add --name "リマインダー" --at 20m --message "買い物リストをまとめて"

# 毎日朝9時（JST）に実行
openclaw cron add --name "朝のまとめ" --cron "0 9 * * *" --tz Asia/Tokyo \
  --message "今日のニュースと天気をまとめて"

# 1時間おきに実行
openclaw cron add --name "定期チェック" --every 1h \
  --system-event "GitHubの通知を確認して"
```

**Stagger（ジッター）** について一つ。毎時0分のcron式にはデフォルトで最大**5分**（`DEFAULT_TOP_OF_HOUR_STAGGER_MS` = 300,000ms）のランダム遅延が適用される。全ユーザーが同時にAPIを叩く集中を避けるためだ。正確な時刻に実行したい場合は `--exact` で無効化できる。

> 💡 **コラム：なぜStagger（ジッター）が必要か？**
> 「毎時0分に実行」と設定するユーザーが100人いると、全員のリクエストが同じ瞬間にLLM APIに殺到する。これは「Thundering Herd（雷の群れ）」と呼ばれるパターンで、APIレート制限に引っかかったり、応答が遅くなったりする原因になる。最大5分のランダム遅延をデフォルトで入れることで、この問題を回避している。分散システムの古典的な知恵だ。

### 9.2.3 ペイロード — 何を実行するか

スケジュールの次は「何をやるか」の指定だ。2種類のペイロードがあり、こちらも**1つだけ**指定が必須だ。

#### `--system-event <text>`（systemEvent）

テキストをシステムイベントとしてメインセッションに注入する。sessionTargetは `main` のみ。Heartbeatがこのイベントを拾って処理する流れになる。メインセッションのコンテキスト（会話履歴、過去の記憶）を参照して処理してほしいタスクに向いている。

#### `--message <text>`（agentTurn）

隔離セッションでエージェントターンを直接開始する。sessionTargetは `isolated` / `current` / `session:<id>` のいずれかだ。独立して完結するタスクに向いている。

agentTurnには追加オプションがある：

- `--model <model>` — 使用モデルの指定（例: `anthropic/claude-sonnet-4-20250514`）
- `--thinking <level>` — 思考レベル（`off` / `minimal` / `low` / `medium` / `high` / `xhigh`）
- `--light-context` — 軽量ブートストラップコンテキストを使用（ワークスペースファイルの読み込みを最小化）

定期実行するタスクでは **`--thinking off --light-context` の組み合わせがコスト抑制に効果的**だ。毎時実行のような高頻度ジョブでは、この設定だけでトークン消費を大幅に削減できる。

```bash
# systemEvent: メインセッションに注入（Heartbeat経由で処理）
openclaw cron add --name "日報リマインダー" \
  --cron "0 18 * * 1-5" --tz Asia/Tokyo \
  --system-event "日報をまとめてDiscordに投稿して"

# agentTurn: 隔離セッションで直接実行（コスト抑制オプション付き）
openclaw cron add --name "朝のニュース" \
  --cron "0 8 * * *" --tz Asia/Tokyo \
  --message "今日のテックニュースを5本ピックアップしてまとめて" \
  --thinking off --light-context \
  --announce --channel discord
```

> 💬 **こうじの実感：** 使い分けの目安は「メインの会話の流れに乗せたい → systemEvent」「独立した結果だけほしい → message」。最初は全部messageでOK。systemEventは上級者向け。

### 9.2.4 セッションターゲット

ペイロードの種類によって使えるセッションターゲットが決まっている。

| ターゲット | ペイロード | 説明 |
|-----------|-----------|------|
| `main` | systemEventのみ | メインセッションにイベント注入 |
| `isolated` | agentTurnのみ | 自動生成の隔離セッション（**デフォルト**） |
| `current` | agentTurnのみ | 現在のセッション。チャット中にその場で `cron add` した場合に、そのセッションで実行したいときに使う |
| `session:<id>` | agentTurnのみ | 指定セッション。特定の長期プロジェクトセッションにタスクを送り込みたい場合に使う |

**最初は `isolated`（デフォルト）で十分だ。** `current` や `session:<id>` は特定のユースケースがある上級者向けなので、必要を感じるまで気にしなくていい。デフォルトは `agentTurn` → `isolated`、`systemEvent` → `main` だ。隔離セッションは自動でフレッシュネス評価が行われ、古くなったら新しいセッションが生成される。セッションラベルは `Cron: <ジョブ名>` で自動設定されるので、後から実行結果を追跡しやすい。

### 9.2.5 配信（Delivery） — 結果をどう届けるか

Cronジョブの実行結果をどう届けるか。3つの配信モードがある。

| モード | 説明 | 設定方法 |
|--------|------|---------|
| `none` | 配信なし | `--no-deliver` |
| `announce` | チャットにサマリー投稿 | `--announce`（isolatedのデフォルト） |
| `webhook` | Webhook URLにPOST | `delivery.mode: "webhook"` + `delivery.to: "<URL>"` を指定 |

```bash
# Discordに結果を送信
openclaw cron add --name "日次レポート" \
  --cron "0 9 * * *" --tz Asia/Tokyo \
  --message "昨日の活動サマリーを作成して" \
  --announce --channel discord

# 配信なし（実行のみ）
openclaw cron add --name "裏方タスク" \
  --every 6h --message "古いログを整理して" --no-deliver
```

`--channel <channel>` で配信チャンネルを、`--to <dest>` で配信先（E.164番号、DiscordチャンネルID等）を指定できる。`--best-effort-deliver` を付けると、配信が失敗してもジョブ自体は失敗扱いにならない。

### 9.2.6 Wake Mode — 結果をいつ処理するか

Cronジョブの実行後、結果をいつ処理するかを `wakeMode`（`--wake <mode>`）で制御する。これはCronジョブ共通の設定であり、特にsystemEventペイロードで重要な意味を持つ。

| モード | 説明 |
|--------|------|
| `now`（デフォルト） | 即座にHeartbeatをトリガーして結果処理 |
| `next-heartbeat` | 次回のHeartbeatサイクルまで遅延 |

ほとんどのケースで `now` を使えばOKだ。`next-heartbeat` は低優先度のタスクで、Heartbeatに「ついでに」処理させたい場合に使う。

### 9.2.7 CLIコマンド一覧

Cronジョブの管理に使うCLIコマンドをまとめる。

```bash
# ジョブ管理
openclaw cron list                    # 一覧表示
openclaw cron list --all              # 無効ジョブも含む
openclaw cron add ...                 # ジョブ追加（エイリアス: create）
openclaw cron edit <id> ...           # ジョブ編集（パッチ形式）
openclaw cron enable <id>             # 有効化
openclaw cron disable <id>            # 無効化
openclaw cron rm <id>                 # 削除（エイリアス: remove, delete）

# デバッグ・運用
openclaw cron status                  # スケジューラーの状態
openclaw cron run <id>                # 即時実行（テスト用）
openclaw cron run <id> --due          # 期限到達時のみ実行
openclaw cron runs --id <id>          # 実行履歴（デフォルト50件）
openclaw cron runs --id <id> --limit 10  # 直近10件
```

`openclaw cron run <id>` はジョブのテスト実行に便利だ。スケジュールを待たずに即座に実行できるので、設定が正しいか確認するのに使える。

### 9.2.8 タイムアウトとリトライ

Cronジョブが暴走したり、一時的なエラーで失敗したりした場合の安全装置だ。**デフォルトで十分な安全装置が入っているので、最初はカスタマイズ不要**。問題が起きたときに参照すればOKだ。

**タイムアウト:**

| 種類 | デフォルト |
|------|-----------|
| systemEventジョブ | 10分（`DEFAULT_JOB_TIMEOUT_MS` = 600,000ms） |
| agentTurnジョブ | 60分（`AGENT_TURN_SAFETY_TIMEOUT_MS` = 3,600,000ms） |
| カスタム | `--timeout-seconds` で指定 |

`--timeout-seconds 0` でタイムアウトを無効化することも可能だが、暴走リスクには注意が必要だ。

**リトライ（ワンショットジョブの一時的エラー時）:**

一時的なエラー（レート制限、ネットワークエラー、タイムアウト、5xxエラー等）に対しては自動リトライが行われる。

- 最大 **3回** リトライ（`DEFAULT_MAX_TRANSIENT_RETRIES`）
- バックオフスケジュール: **30秒 → 1分 → 5分**（内部的には5段階のスケジュール `[30秒, 1分, 5分, 15分, 60分]` から先頭3段階が使われる。カスタマイズ時は `cron.retry.backoffMs` で全段階を指定可能）

**失敗アラート:**

連続してエラーが発生すると、アラートで通知される。これらの安全装置の全体像は次章（セキュリティとアクセス制御）で詳しく扱う。

- 連続 **2回** エラー後にアラート通知（`DEFAULT_FAILURE_ALERT_AFTER`）
- アラート間隔: 最小 **60分**（`DEFAULT_FAILURE_ALERT_COOLDOWN_MS`）
- `openclaw cron edit <id> --failure-alert` / `--no-failure-alert` でジョブ単位でトグル可能（作成後に `edit` で設定）

### 9.2.9 ジョブライフサイクル

Cronジョブが追加されてから結果が届くまでの全体フローだ。

<!-- IMAGE: ch09-cron-lifecycle.png - Cronジョブのライフサイクル全体図。add→スケジュール→実行→結果処理→配信→次回スケジュール。エラー時のリトライ分岐も含む -->

```
add → スケジュール登録 → 時刻到達 → 実行
  ├── 成功 → 配信 → 状態更新 → (ワンショット+deleteAfterRun → 削除)
  ├── エラー → リトライ判定 → 失敗アラート判定
  └── スキップ → 状態更新
  → wakeMode に応じて Heartbeat トリガー
```

ワンショットジョブ（`at`）はデフォルトでは成功後もジョブ定義が保持される。`--delete-after-run` を付けると成功後に自動削除される。

---

## 9.3 Webhook — 外部イベントでエージェントを起こす

HeartbeatとCronは「時間駆動」——こちらが決めたタイミングで動く仕組みだった。Webhookは「イベント駆動」だ。外部サービスの変化をリアルタイムにキャッチして、エージェントを起こす。

### 9.3.1 Webhookの2つのレイヤー

OpenClawのWebhookは2つの異なる仕組みで構成されている。

| レイヤー | 用途 | 例 |
|---------|------|-----|
| **Plugin Webhook Ingress** | チャンネルプラグインの受信用 | BlueBubbles等 |
| **Hooks** | 外部サービス統合 | Gmail、カスタムWebhook |

Plugin Webhook Ingressはチャンネルプラグインの範疇（第7章参照）なので、この章では主に**Hooks**（外部サービス統合）を解説する。

### 9.3.2 Hooksの仕組み

Hooksは `config.yaml` で設定する。

```yaml
# config.yaml のHooks設定
hooks:
  enabled: true
  path: "/hooks"
  token: "your-secret-token"
  mappings:
    - wakeMode: "now"          # 即座にHeartbeat実行
    # or
    - wakeMode: "next-heartbeat" # 次回Heartbeatで処理
```

Hookがリクエストを受信すると、`wakeMode` に応じてHeartbeatをトリガーする。トークンベースの認証で不正アクセスを防止する仕組みだ。

`hooks.enabled` はデフォルトで無効になっている。ヘルプテキストにも "Keep disabled unless you are actively routing" とあるように、使うときだけ明示的に有効化すること。

### 9.3.3 Gmail連携 — 実践的なWebhookの使い方

OpenClawにはGmail連携のための専用CLIが用意されている。これがWebhookの最も実践的なユースケースだ。

```bash
# Gmail Webhookのセットアップ（ウィザード形式）
openclaw webhooks gmail setup \
  --account your@gmail.com \
  --project your-gcp-project \
  --tailscale funnel

# Watch + 自動更新ループの実行
openclaw webhooks gmail run
```

#### セットアップの流れ

1. GCPプロジェクトでPub/Subトピック・サブスクリプションを作成
2. Gmail APIのWatch設定（INBOXの変更を監視）
3. Tailscale Funnel（またはServe）で外部からアクセス可能にする
4. OpenClawのHooksエンドポイントに接続

<!-- IMAGE: ch09-gmail-webhook-flow.png - Gmail → GCP Pub/Sub → Tailscale Funnel → OpenClaw Gateway → Heartbeat → エージェント処理 の流れ図 -->

#### 主要オプション

| オプション | デフォルト | 説明 |
|-----------|-----------|------|
| `--label <label>` | `INBOX` | 監視するGmailラベル |
| `--tailscale <mode>` | `funnel` | 公開方式（`funnel` / `serve` / `off`） |
| `--include-body` | `true` | メール本文スニペットを含むか |
| `--max-bytes <n>` | 20,000 | 本文最大バイト数 |
| `--renew-minutes <n>` | 720（12時間） | Watch更新間隔 |

Tailscale Funnelを使えば、インターネットに直接Webhookエンドポイントを晒さずにGCPからのPush通知を受け取れる。セキュリティ面でも安心だ。

> 💬 **こうじの実感：** Gmail連携はGCPの設定が一番面倒。Pub/Subのトピック作って、サブスクリプション作って…ってやってると「本当にこれ動くの？」って不安になる。でも一回セットアップしちゃえば `openclaw webhooks gmail run` で放置できるから、最初の30分だけ我慢。動いた瞬間の「おおっ！」は格別だよ。

### 9.3.4 Webhookのセキュリティ

Webhookエンドポイントを外部に公開する以上、セキュリティは重要だ。OpenClawはいくつかの防御レイヤーを持っている。

- **レート制限**: 1分あたり **120リクエスト**（デフォルト）
- **ボディサイズ制限**: 認証後 **1MB** / 認証前 **64KB**
- **In-Flight制限**: 同一キーで最大 **8並行リクエスト**
- **パス正規化**: ディレクトリトラバーサル攻撃（`../` 等）を防止
- **異常検知**: 400/401/408/413/415/429 を追跡し、25件ごとにログ出力

> ⚠️ **ここに注意**: Webhookエンドポイントを公開する場合は、必ずトークン認証を設定すること。Tailscale Funnelを使えばインターネットに直接晒さずに済むので、可能であれば Funnel 経由を推奨する。これらのセキュリティ機構の全体像は次章で詳しく扱う。

---

## 9.4 配信キュー — 裏側の仕組み

CronやHeartbeatの結果はどうやってDiscordやTelegramに届くのか。その裏側には**配信キュー**という仕組みがある。地味なセクションだが、OpenClawの堅牢さを支える重要なパーツだ。

### 9.4.1 配信の流れ

エージェントの実行が完了すると、結果は配信キューに入る。

```
エージェント実行完了 → enqueueDelivery()
  ↓
{id}.json として永続化（<stateDir>/delivery-queue/）
  ↓
配信試行
  ├── 成功 → {id}.json → {id}.delivered → 削除（2段階）
  └── 失敗 → retryCount++ → バックオフ待機 → 再試行
       └── 5回失敗 → failed/ ディレクトリに移動
```

注目すべきは**2段階削除**だ。まず `{id}.json` を `{id}.delivered` にリネームし、その後に削除する。この2ステップにより、途中でクラッシュしても再送を防止できる。`.delivered` マーカーが残っていれば「配信済み」と判断できるからだ。

> 💡 **コラム：なぜ2段階削除なのか？**
> 配信完了 → ファイル削除、を1ステップでやると「配信成功の直後、削除前にプロセスがクラッシュ」したときに問題が起きる。再起動後、ファイルが残っているので未配信と判断して**再送**してしまう。2段階削除（リネーム → 削除）なら、リネーム後にクラッシュしても `.delivered` マーカーで配信済みだとわかる。データベースのトランザクションに似た考え方だ。

### 9.4.2 リトライとバックオフ

配信に失敗した場合、以下のスケジュールでリトライされる。

- バックオフ: **5秒 → 25秒 → 2分 → 10分**（4段階。5回目以降は10分間隔を維持）
- 最大リトライ: **5回**（超過で `failed/` に移動）
- バックオフ期間中のエントリは配信試行をスキップ

### 9.4.3 永久エラー — リトライしても無駄なケース

すべてのエラーがリトライの対象ではない。以下のエラーはリトライせず即座に `failed/` に移動する。

- `chat not found` / `user not found`
- `bot was blocked by the user`
- `outbound not configured for channel`
- etc.

> 💬 **こうじの実感：** 「botがブロックされた」のにリトライしても意味ないもんね。この辺の割り切りがちゃんとしてるのは安心感ある。無限リトライで余計な負荷がかかることもないし。

### 9.4.4 Gateway再起動時のリカバリー

Gatewayを再起動すると、起動時に `recoverPendingDeliveries()` が未配信エントリを処理する。

- 最大 **60秒** の時間予算で古い順に処理
- `.delivered` マーカーがあれば即クリーンアップ（再送しない）
- 時間予算を超えた分は次回起動に延期

大量の未配信メッセージが溜まっていると、起動時に最大60秒かかる可能性がある。その間、新しい配信は少し待たされるかもしれないが、メッセージが失われることはない。

---

## 9.5 実践レシピ

3つの自動化手段を理解したところで、具体的なユースケースをレシピ形式でまとめる。各レシピで「なぜその自動化手段を選んだか」も明示しているので、自分のケースに当てはめる参考にしてほしい。

### レシピ1: 毎朝のニュース＆天気まとめ（Cron + agentTurn）

```bash
openclaw cron add \
  --name "朝のブリーフィング" \
  --cron "0 8 * * *" --tz Asia/Tokyo \
  --message "今日のテックニュース5本と東京の天気をまとめて。簡潔に、箇条書きで。" \
  --thinking off --light-context \
  --announce --channel discord
```

- **なぜCron？** → 毎朝決まった時間に実行したいから
- **なぜagentTurn？** → メインセッションのコンテキストは不要。独立したタスク
- **なぜannounce？** → 結果をDiscordで受け取りたいから
- **コスト抑制:** `--thinking off --light-context` で毎日のトークン消費を最小化

### レシピ2: Gmailの重要メール通知（Webhook + Heartbeat）

```bash
# 1. Gmail Webhookセットアップ
openclaw webhooks gmail setup \
  --account your@gmail.com \
  --project my-gcp-project \
  --tailscale funnel

# 2. HEARTBEAT.md にGmail処理の指示を追加
echo "- [ ] 新しいGmail通知があれば、送信者と件名をDiscordで教えて" >> HEARTBEAT.md
```

- **なぜWebhook？** → メール受信という外部イベントに反応したいから
- **Webhook → Heartbeat → 処理 → 配信** の連携パターン

### レシピ3: 週次レポートの自動生成（Cron + agentTurn + 配信）

```bash
openclaw cron add \
  --name "週次レポート" \
  --cron "0 17 * * 5" --tz Asia/Tokyo \
  --message "今週のmemory/ファイルを確認して、週次活動レポートを作成。memory/weekly/に保存して。" \
  --announce --channel discord \
  --timeout-seconds 300
```

- **なぜ金曜17時？** → 週末前にまとめを出したいから
- **タイムアウト延長**: レポート生成は時間がかかるため5分（300秒）に設定

### レシピ4: 定期ヘルスチェック（Cron + systemEvent + Heartbeat）

```bash
openclaw cron add \
  --name "ヘルスチェック" \
  --every 6h \
  --system-event "サーバーのディスク使用量とメモリ使用量を確認して。異常があればDiscordで報告。" \
  --wake now
```

- **なぜsystemEvent？** → メインセッションのコンテキスト（過去の異常履歴等）を参照したいから
- **なぜevery 6h？** → cron式ほど細かい制御は不要。シンプルな定期実行

<!-- IMAGE: ch09-recipe-decision-tree.png - 「どの自動化手段を使うか」の判断フローチャート。定期？→はい→時間指定？→Cron / いいえ→Heartbeat / 外部イベント？→Webhook -->

> 💬 **こうじの実感：** レシピ1の「朝のブリーフィング」は自分でも毎日使ってるやつ。朝起きてDiscord開くと、もうニュースと天気がまとまってる。これだけでも「自動化やってよかった」って思える。

---

## 9.6 まとめ — 3つの自動化手段の使い分け

### 使い分けチャート

| 質問 | Heartbeat | Cron | Webhook |
|------|-----------|------|---------|
| **いつ動く？** | 定期間隔（デフォルト30分） | 指定した時間/間隔 | 外部イベント発生時 |
| **何をする？** | HEARTBEAT.mdの指示に従う | 指定したメッセージ/イベントを実行 | イベントをHeartbeatに伝達 |
| **設定の手軽さ** | ★★★（config.yaml + HEARTBEAT.md） | ★★☆（CLIコマンド） | ★☆☆（外部サービス設定が必要） |
| **典型的な用途** | 定期巡回、未読チェック | 時間指定タスク、リマインダー | メール通知、GitHub連携 |
| **メインセッションへの影響** | あり（デフォルト） | 選択可能（main/isolated） | Heartbeat経由（間接） |

### 判断フロー

迷ったときはこの順番で考えてみてほしい。

1. 「外部サービスの変化に反応したい」→ **Webhook**
2. 「決まった時間に決まったことをやりたい」→ **Cron**
3. 「定期的にやることリストを確認してほしい」→ **Heartbeat**
4. 「とりあえず自動化してみたい」→ **まずHeartbeatから始めよう**

Heartbeatは設定が簡単で効果を実感しやすいので、最初の一歩として最適だ。慣れてきたらCronで時間指定のタスクを追加し、必要に応じてWebhookで外部サービスとの連携を組み込む——段階的に自動化の範囲を広げていくのがおすすめだ。

<!-- IMAGE: ch09-automation-summary.png - 3つの自動化手段の使い分けを視覚的にまとめた図。時間軸（定期/指定/イベント）と複雑さ軸で配置 -->

### 次章への橋渡し

エージェントに自律性を与えた。Heartbeatで定期巡回し、Cronで時間指定タスクをこなし、Webhookで外部イベントに即応する——もはや寝ている間も働いてくれるパートナーだ。

しかし、ここで一つ立ち止まって考えたい。**自律的に動くからこそ、安全装置が必要**なのだ。暴走したらどうする？権限を越えた操作をしたら？次章では、OpenClawの「セキュリティとアクセス制御」を学ぶ。エージェントを信頼するために、まず制御する方法を知ろう。

---

## 付録：設定リファレンス

### config.yaml の自動化関連設定キー一覧

主要な設定項目をカテゴリ別にまとめた早見表だ。

**Cron関連:**

| キー | デフォルト | 説明 |
|-----|-----------|------|
| `cron.enabled` | `true` | スケジューラー有効化 |
| `cron.store` | 自動 | ジョブストアのファイルパス |
| `cron.maxConcurrentRuns` | `1` | 同時実行数上限 |
| `cron.sessionRetention` | `"24h"` | セッション保持期間 |
| `cron.retry.maxAttempts` | `3` | 一時的エラーの最大リトライ回数 |
| `cron.retry.backoffMs` | `[30000, 60000, 300000]` | リトライバックオフ（ms） |
| `cron.retry.retryOn` | 全種別 | リトライ対象エラー種別の限定 |
| `cron.runLog.maxBytes` | `2000000`（2MB） | ジョブログの最大サイズ |
| `cron.runLog.keepLines` | `2000` | maxBytes超過時の保持行数 |
| `cron.failureAlert.after` | `2` | N回連続エラー後にアラート |
| `cron.failureAlert.cooldownMs` | `3600000`（60分） | アラート間の最小間隔 |

**Heartbeat関連:**

| キー | デフォルト | 説明 |
|-----|-----------|------|
| `agents.defaults.heartbeat.every` | `"30m"` | 間隔 |
| `agents.defaults.heartbeat.target` | `"none"` | 配信先 |
| `agents.defaults.heartbeat.prompt` | 組み込みプロンプト | Heartbeatプロンプトのカスタマイズ |
| `agents.defaults.heartbeat.model` | — | モデルオーバーライド（上級） |
| `agents.defaults.heartbeat.ackMaxChars` | `300` | HEARTBEAT_OK短文上限 |
| `agents.defaults.heartbeat.isolatedSession` | `false` | 隔離セッションでの実行 |
| `agents.defaults.heartbeat.activeHours.*` | — | 静粛時間 |

**Hooks関連:**

| キー | デフォルト | 説明 |
|-----|-----------|------|
| `hooks.enabled` | `false` | 有効化 |
| `hooks.path` | — | エンドポイントパス |
| `hooks.token` | — | 認証トークン |
| `hooks.mappings` | — | Hookルーティング設定（`id`, `match`, `action`, `wakeMode` 等を含む配列） |
| `hooks.maxBodyBytes` | — | リクエストボディの最大サイズ |

**環境変数:**

| 変数 | 説明 |
|------|------|
| `OPENCLAW_SKIP_CRON=1` | Cronスケジューラー無効化 |
