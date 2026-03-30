# 第7章：チャンネル接続

> _"We are all connected."_ — 映画『マトリックス』で語られた、全てがつながった世界の真理（っぽいやつ）。OpenClawも同じだ。Discord、Telegram、WhatsApp、Slack……どのプラットフォームからでも、同じエージェントと話している感覚を与える。第5章で学んだセッションキーに `discord:direct:` や `telegram:group:` が含まれていたのを覚えているだろうか？あれこそが、この「接続」の正体だった。チャンネルは、OpenClawが外部世界と出会うための「言語の翻訳者」なのだ。

---

## この章が解き明かす「つながりの仕組み」

第5章でセッション管理を、第6章でコンテキストエンジンを学んだあなたは、OpenClawの「内側」をよく理解している。しかし、外部世界との接点——Discord、WhatsApp、Telegram——がどう動作しているかは、まだ「魔法」のように見えているはずだ。

この章のテーマは**チャンネル接続**——OpenClawが20+のメッセージングプラットフォームと対話する仕組みの全貌だ。アーキテクチャ、ルーティング、設定方法、セキュリティ……すべてを解剖する。

読み終える頃には、「Discord経由でメッセージを送ると、どうしてCLIでも同じエージェントの記憶を参照できるのか」が完全にわかるようになる。そして、独自プラットフォームへの対応方法も身につく。チャンネルは魔法じゃない。論理的で拡張可能なシステムなのだ。

<!-- IMAGE: channel-overview.png - チャンネルがユーザー・プラットフォーム・Gateway・エージェント間の翻訳者として機能する全体図 -->

---

## 7.1 チャンネルとは何か — 外部世界への接続口

チャンネルとは、OpenClawにおける**メッセージングプラットフォームへの接続インターフェース**だ。より具体的に言えば、Discord Bot API、Telegram grammY、WhatsApp Baileysライブラリといった「プラットフォーム固有のAPI」と、OpenClaw内部の「統一されたメッセージ形式」を相互変換する層である。

> 💬 **こうじの実感：** 実際にDiscordでOpenClawを使ってみると、「チャンネル」の威力がよくわかる。Discord特有のembedやリアクション機能も自然に使えるし、同じエージェントとTelegramでも会話できる。プラットフォームの違いを意識せずに済むのは、この「翻訳者」のおかげなんだなと実感する。

### 翻訳者としてのチャンネル

チャンネルを「翻訳者」だと考えるとわかりやすい。

あなたがWhatsAppで「今日の天気は？」と送ると：

1. **WhatsAppサーバー** → Baileysライブラリがメッセージを受信
2. **WhatsAppチャンネル** → 送信者ID、時刻、メディア等をOpenClaw内部形式に変換
3. **Gateway** → セッション管理・ルーティング経由でエージェントに配信
4. **エージェント** → 天気情報を取得、返答を生成
5. **WhatsAppチャンネル** → OpenClaw内部形式をWhatsApp API形式に変換
6. **WhatsAppサーバー** → あなたのアプリに天気情報が届く

この過程で、チャンネルは「WhatsApp語」と「OpenClaw語」を双方向に翻訳している。Discord、Telegram、Slackでも基本的な流れは同じだが、プラットフォームごとの「方言」（APIの違い、認証方式、メディア対応等）を各チャンネルが吸収している。

### 第5章との関係：セッションキーの再来

第5章で学んだセッションキー `agent:main:discord:direct:990665145992745030` を思い出そう。この `:discord:` 部分こそが、チャンネルの正体だった。

- `discord` — チャンネル名（どのプラットフォーム経由か）
- `direct` — チャットタイプ（DM、グループ、チャンネル）
- `990665...` — 送信者ID（プラットフォーム固有の識別子）

つまり、セッション管理とチャンネル接続は表裏一体だ。チャンネルが「どこから来たか」を記録し、セッション管理が「どこに戻すか」を制御する。

### アーキテクチャ上の位置

チャンネルがOpenClawの全体アーキテクチャのどこに位置するかを確認しよう：

```
ユーザー ↔ プラットフォーム ↔ チャンネルプラグイン ↔ Gateway ↔ エージェント
```

| 層 | 役割 |
|----|----|
| ユーザー | Discord、WhatsApp等のアプリを使う人 |
| プラットフォーム | Discord、Telegram、WhatsApp等のサービス |
| チャンネルプラグイン | プラットフォーム固有APIとOpenClaw内部形式の変換 |
| Gateway | セッション管理・ルーティング・エージェント配信 |
| エージェント | AI推論・ツール実行・応答生成 |

チャンネルはGatewayの「手下」として動作し、各プラットフォームの「出張所」を運営している感覚だ。

<!-- IMAGE: platform-categories.png - 6つのカテゴリーごとにプラットフォームを分類した一覧図 -->

---

## 7.2 対応プラットフォーム一覧 — 20以上の接続先を分類整理

**どれから始めればいいかわからない？→ Telegram一択**

OpenClawは2026年3月時点で、20以上のメッセージングプラットフォームに対応している。初心者は迷わずTelegramから始めることを強く推奨する。Bot Token取得だけで3分で設定完了し、すべてのOpenClaw機能を体験できる。

これらを機能と設定難易度で分類してみよう。

### 6つのカテゴリー分類

| カテゴリー | プラットフォーム | 特徴 |
|------------|------------------|------|
| **メッセージング** | WhatsApp、Telegram、Signal、LINE、Zalo | 個人間コミュニケーション重視 |
| **チャット** | Discord、Slack、Microsoft Teams、Google Chat | チーム・組織内コミュニケーション |
| **プロトコル系** | Matrix、IRC、Nostr | オープンプロトコル・分散化 |
| **Apple生態系** | iMessage (legacy)、BlueBubbles | macOS・iOS統合 |
| **エンタープライズ** | Mattermost、Nextcloud Talk、Synology Chat | 自社運営・プライベートクラウド |
| **その他** | WebChat、Twitch、Feishu、Tlon、Voice Call | 特殊用途・実験的対応 |

### 設定の簡単さランキング

実際に使い始めるときの参考に、設定の簡単さでランキングを付けてみる：

| 順位 | プラットフォーム | 設定時間 | 必要な作業 |
|------|------------------|----------|------------|
| 🥇 | Telegram | 3分 | Bot Token取得のみ |
| 🥈 | Discord | 5分 | アプリ作成、特権インテント有効化 |
| 🥈 | Slack | 5分 | アプリ作成、Socket Mode有効化 |
| 🥉 | WhatsApp | 10分 | QRペアリング、状態管理の理解 |
| 4位 | Signal | 15分 | signal-cli セットアップ |
| 5位 | iMessage (BlueBubbles) | 30分 | macOS server構築 |

> 💬 **こうじの実感：** 個人的にはTelegramが一番楽だった。@BotFatherとの会話だけでBot作成完了、トークンをコピペするだけ。Discordは機能豊富だけど、特権インテント設定で初心者がつまずきがち。WhatsAppは実用性が高いが、QRペアリングの概念理解が必要。迷ったらTelegramで基本を覚えてから他に展開するのが賢明。

### プラグイン vs コアエクステンション

重要な区別として、プラットフォーム対応には2つの実装レベルがある：

**コアエクステンション（OpenClaw本体にバンドル）：**
- Discord、Telegram、WhatsApp、Slack、IRC、WebChat
- `/usr/lib/node_modules/openclaw/dist/extensions/` に含まれる
- 追加インストール不要

**プラグイン（外部パッケージとして配布）：**
- LINE (`@openclaw/line`)、Matrix (`@openclaw/matrix`)、Microsoft Teams等
- `openclaw plugins install @openclaw/<name>` でインストール
- 特定用途・ニッチな需要に対応

最初の一歩として、コアエクステンションのどれかから始めることを強く推奨する。

> 💡 **コラム：なぜチャンネルは「プラグイン」なのか**  
> OpenClawがチャンネルをプラグイン化したのは、3つのメリットがあるからだ。まず「**新しいプラットフォームの追加**」——新SNSが登場しても、コア機能を変更せずプラグインとして対応できる。次に「**独立したアップデート**」——Telegram API仕様変更があっても、そのプラグインだけアップデートすればいい。最後に「**コア機能との分離**」——チャンネル障害がGateway全体を巻き込まない。これにより、外部開発者も独自プラットフォーム対応を作成・配布できるエコシステムが生まれている。

<!-- IMAGE: channel-architecture.png - コア機能とプラグイン提供機能の役割分担を示すレイヤー図 -->

---

## 7.3 アーキテクチャ — プラグインシステムとの関係

チャンネルがOpenClawのどこに実装されているか、その設計思想を深掘りしてみよう。チャンネルは「専門店」のような設計になっている。

### コア vs プラグインの役割分担

OpenClawのチャンネルアーキテクチャは、「コア機能」と「プラグイン提供機能」を明確に分離している：

**コア提供（Gateway担当）：**
- 共通`message`ツール（送信インターフェース）
- セッション管理・ルーティング
- プロンプト配線・ディスパッチ
- セキュリティフレームワーク

**プラグイン提供（各チャンネル担当）：**
- アカウント解決・設定ウィザード
- セキュリティ（DMポリシー・許可リスト）
- ペアリング（DM承認フロー）
- アウトバウンド（プラットフォーム送信）
- スレッディング（返信の配線方法）

これは「専門店モデル」だと考えるとわかりやすい。Gatewayは「デパートの管理会社」で、各チャンネルは「テナントの専門店」だ。共通のインフラ（エレベーター、警備、会計システム）はデパート側が提供し、専門的なサービス（商品、接客、店舗運営）は各テナントが担当する。

### ディレクトリ構造

実際のファイル配置を見ると、この設計がよくわかる：

```
/usr/lib/node_modules/openclaw/dist/extensions/
├── discord/
│   ├── package.json      # "openclaw.channel" メタデータ
│   ├── channel.js        # Discord Bot API実装
│   └── setup.js          # 設定ウィザード
├── whatsapp/
│   ├── package.json      # Baileysライブラリ構成
│   ├── channel.js        # WhatsApp Business API実装
│   └── setup.js          # QRペアリング処理
├── telegram/
│   ├── package.json      # grammY構成
│   ├── channel.js        # Bot API実装
│   └── setup.js          # Bot Token設定
└── slack/
    ├── package.json      # Bolt SDK構成
    ├── channel.js        # Socket Mode実装
    └── setup.js          # アプリトークン設定
```

各チャンネルは独立したnpmパッケージとして構成され、`package.json` に `"openclaw.channel"` フィールドを持つことでOpenClawに認識される。この設計により、チャンネル単体でのアップデート・配布・テストが可能になっている。

### メッセージツール統合の妙

特筆すべきは、チャンネルプラグインが「独自のsend/edit/reactツール」を持たないことだ。代わりにOpenClaw中核が共通 `message` ツールを提供し、各チャンネルはその「アウトバウンド処理」のみを担当する。

これにより、エージェント側は「どのプラットフォームに送信するか」を意識せずに済む。 `message` ツールで送信すれば、現在のセッションキーに基づいてGatewayが自動的に適切なチャンネルにルーティングする。「翻訳者」の真価がここにある。

<!-- IMAGE: routing-decision-tree.png - 8段階のエージェント選択ルールを決定木で表現した図 -->

---

## 7.4 チャンネルルーティング — メッセージはどうエージェントに届くか

ここからがチャンネル接続の核心だ。メッセージが「どのエージェントに」「どのセッションで」処理されるかは、決定論的なルールに従って決まる。感覚ではなく、論理だ。

### ルーティング原則

OpenClawのルーティングは**「メッセージが来たチャンネルに戻す」**という単純な原則に基づく。これは「住所配達」のようなもので、AIモデルがチャンネルを選択するのではなく、ホスト設定によって決定論的に決まる。

つまり、Discord DMで質問すればDiscord DMに回答が返り、Telegram グループで質問すればそのグループに回答が返る。これにより、会話の文脈が散らばることなく、プラットフォーム横断でも一貫した体験を提供できる。

### よく使うパターン3つ

実際の運用では、以下3つのパターンが使用頻度の大半を占める：

1. **DM（個人チャット）** — Discord・Telegram・WhatsAppでの1対1会話
2. **グループチャット** — `@エージェント名` でのメンション応答
3. **専用チャンネル** — Discord・Slackの専用部屋での常時応答

これらを理解すれば、日常的な使用には十分対応できる。

### エージェント選択の8段階ルール

メッセージを受信したとき、どのエージェントが処理するかは以下の8段階で決定される：

| 優先順位 | 判定条件 | 具体例 |
|----------|----------|--------|
| 1 | **完全一致** | `bindings`の`peer.kind` + `peer.id`が完全マッチ |
| 2 | **親ピア一致** | スレッド継承（返信チェーン） |
| 3 | **Guild + roles一致** | Discord: `guildId` + `roles`の組み合わせ |
| 4 | **Guild一致** | Discord: `guildId`のみ |
| 5 | **チーム一致** | Slack: `teamId` |
| 6 | **アカウント一致** | チャンネル上の`accountId` |
| 7 | **チャンネル一致** | チャンネル上の任意アカウント（`accountId: "*"`） |
| 8 | **デフォルトエージェント** | `agents.list[].default`、なければ最初のエントリ、最終的には`main` |

この評価は上から順に行われ、最初にマッチした条件でエージェントが決定される。

### セッションキー形式の再確認

第5章で学んだセッションキー形式を、チャンネル視点で再整理してみよう：

```
# 基本形式
agent:<agentId>:<channel>:<type>:<id>

# 実際の例
agent:main:telegram:group:-1001234567890           # Telegramグループ
agent:main:discord:channel:123456                  # Discordチャンネル
agent:main:whatsapp:direct:+15551234567            # WhatsApp DM

# スレッド拡張
agent:main:discord:channel:123456:thread:987654    # Discordスレッド
agent:main:telegram:group:-1001234567890:topic:42  # Telegramフォーラムトピック
```

このキー生成を担当するのが各チャンネルプラグインの `resolveSessionKey()` 実装だ。プラットフォームごとの識別子の違い（Discordのsnowflake ID（Discord独自の一意識別番号）、Telegramのマイナス ID（Telegramがグループに付与する負のID）等）を吸収し、統一的なキー形式に変換する。

### メインDMルート固定の仕組み

第5章で触れた「メインDMルート固定」も、チャンネルルーティングと密接に関係している。`session.dmScope`が`main`の時、DMは一つのメインセッション（`agent:main:main`）を共有する。

重要なのは、`allowFrom`に単一エントリしかない場合、所有者以外のDMは`lastRoute`を上書きしないよう保護されることだ。これにより、所有者のメインセッションが他人のメッセージで「汚染」されることを防いでいる。

<!-- IMAGE: group-policy-flow.png - groupPolicyから最終応答までの評価フローチャート -->

---

## 7.5 グループチャット対応 — 複数人の会話での振る舞い

グループチャットでのAI参加は「パーティーでの会話参加」に似ている。いつ話すか、誰に向けて話すか、どのタイミングで発言するか——これらすべてがルールで制御されている。

### デフォルト動作：賢い「お客さん」

OpenClawのデフォルト設定は、グループチャットで「お客さん」のように振る舞う：

- **グループ制限：** `groupPolicy: "allowlist"`（許可リストに含まれるグループのみ参加）
- **メンション必須：** 明示的に無効化しない限り@メンション必要
- **DM vs グループ：** DMアクセスは`*.allowFrom`、グループアクセスは`*.groupPolicy` + `*.groups`で制御

つまり、勝手に全てのグループに参加するのではなく、「招待されたパーティー」でかつ「名前を呼ばれた時だけ」発言する。

### メンション検出の2つの仕組み

OpenClawは2つの方法でメンションを検出する：

**1. 自動検出（プラットフォーム提供）：**
- Discord、Slack等の明示的@メンション
- Telegramの返信（Bot メッセージへの返信として検出）
- WhatsAppの引用返信

**2. パターンマッチ（カスタム設定）：**
```json5
{
  agents: {
    list: {
      main: {
        groupChat: {
          mentionPatterns: ["@openclaw", "openclaw", "\\+15555550123"]
        }
      }
    }
  }
}
```

パターンマッチは大文字小文字を区別せず、正規表現として安全に処理される。これにより、「Hey OpenClaw」や「おーい OpenClaw」のような自然な呼びかけにも対応できる。

### グループポリシーの3つのモード

グループへの参加方針は`groupPolicy`で制御される：

| ポリシー | 動作 | 用途 |
|----------|------|------|
| `disabled` | グループメッセージを完全に無視 | 個人専用 |
| `allowlist` | 許可リストのグループのみ参加（デフォルト） | 管理された参加 |
| `open` | 全てのグループで応答可能 | オープンBot |

### 評価順序の重要性

グループメッセージの処理は、以下の順序で評価される：

1. **groupPolicy** — disabled/allowlist/openの判定
2. **グループ許可リスト** — `*.groups`, `*.groupAllowFrom`
3. **メンションゲーティング** — `requireMention`, `/activation`

どの段階でも「参加不可」と判定されれば、それ以降の処理は行われない。一方、「参加可能」と判定されれば、通常のエージェント処理に進む。

### 実用的な設定例

```json5
{
  channels: {
    whatsapp: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],  // グループ管理者のみ招待可能
      groups: {
        "*": { requireMention: true },           // 全グループでメンション必要
        "120363403215116621@g.us": {             // 特定グループは常時応答
          requireMention: false 
        }
      }
    }
  }
}
```

この設定により、管理者が招待したグループでのみ動作し、ほとんどのグループではメンション時のみ応答するが、特定の作業グループでは全てのメッセージに対応する——という柔軟な運用が可能になる。

> 💡 **コラム：位置情報はこう扱われる**  
> Telegram、WhatsApp、Matrixでは位置情報の送受信にも対応している。受信した位置情報は `📍 48.858844, 2.294351 ±12m` という形式でテキスト化されると同時に、`LocationLat`, `LocationLon`, `LocationAccuracy` といった構造化フィールドとしてもエージェントに渡される。live locationの場合は `🛰 Live location: ...` というプレフィックスが付く。これにより、エージェントは「近くのカフェを探して」「この場所の天気を教えて」といったリクエストに正確に答えることができる。

<!-- IMAGE: setup-comparison.png - 4つのプラットフォームの設定難易度を比較したグラフ -->

---

## 7.6 主要チャンネルの設定ガイド — Discord・Telegram・WhatsApp・Slack

理論はここまで。実際に手を動かしてチャンネルを設定してみよう。最も使用頻度の高い4つのプラットフォームの「レシピ」を紹介する。

### Discord：専用サーバー推奨

Discordは最も多機能だが、設定手順がやや複雑だ。専用サーバーを作ることを強く推奨する。

**Step 1: アプリ作成**  
[Discord Developer Portal](https://discord.com/developers/applications) → New Application

**Step 2: Bot設定**  
Bot タブ → Reset Token → トークンをコピー & 保存

**Step 3: 特権インテント有効化（重要）**  
- Message Content Intent（必須）
- Server Members Intent（推奨）

**Step 4: 招待URL生成**  
OAuth2 タブ → bot + applications.commands + 基本的な権限を選択

**Step 5: Developer Mode有効化**  
Discord設定 → 詳細設定 → 開発者モード

**Step 6: ID取得**  
- サーバーID（サーバー名右クリック → Copy Server ID）
- ユーザーID（プロフィール右クリック → Copy User ID）

**設定例：**
```bash
export DISCORD_BOT_TOKEN="YOUR_TOKEN"
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN
```

```json5
{
  channels: {
    discord: {
      enabled: true,
      dmPolicy: "pairing",
      allowFrom: ["990665145992745030"],  // あなたのユーザーID
      guilds: {
        "123456789012345678": {           // サーバーID
          channels: { 
            "general": { allow: true } 
          }
        }
      }
    }
  }
}
```

### Telegram：3分で完了する最速セットアップ

Telegramは最も簡単に設定できるプラットフォームだ。

**Step 1: Bot作成**  
@BotFather にメッセージ → `/newbot` → Bot名とユーザー名を設定

**Step 2: トークン保存**  
BotFatherから受け取ったトークンを保存

**Step 3: 設定**
```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "123456789:ABCdef1234567890abcdef1234567890ABC",
      dmPolicy: "pairing",
      groups: { 
        "*": { requireMention: true } 
      }
    }
  }
}
```

**Step 4: 起動と承認**
```bash
openclaw gateway
# 別ターミナルで
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

### WhatsApp：QRペアリング（専用番号推奨）

WhatsAppはQRコードでのペアリングが特徴的だ。個人番号でも動作するが、専用番号の使用を強く推奨する。

**Step 1: 基本設定**
```json5
{
  channels: {
    whatsapp: {
      enabled: true,
      dmPolicy: "pairing",
      allowFrom: ["+15551234567"],          // あなたの番号
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"]      // グループ招待許可者
    }
  }
}
```

**Step 2: ペアリング開始**
```bash
openclaw channels login --channel whatsapp
```

**Step 3: Gateway起動**
```bash
openclaw gateway
# QRコードが表示される → WhatsAppアプリでスキャン
```

**Step 4: 承認**
```bash
openclaw pairing list whatsapp
openclaw pairing approve whatsapp <8文字コード>
```

**注意事項：**
- ペアリング後の状態管理ファイルを大切に保管
- 接続が不安定になったら `openclaw channels login --channel whatsapp` で再ペアリング

### Slack：Socket Mode設定

SlackはワークスペースアプリとしてSocket Modeで動作する。

**Step 1: アプリ作成**  
[Slack API](https://api.slack.com/apps) → Create New App → From scratch

**Step 2: Socket Mode有効化**  
Settings → Socket Mode → Enable Socket Mode

**Step 3: App Token作成**  
OAuth & Permissions → App-Level Tokens → Generate Token  
Scope: `connections:write`

**Step 4: Bot Token取得**  
Install App → Copy Bot User OAuth Token

**Step 5: イベント購読**  
Event Subscriptions → Subscribe to bot events:  
`app_mention`, `message.channels`, `message.groups`, `message.im`, `reaction_added`, `reaction_removed`

**Step 6: 設定**
```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      appToken: "xapp-1-A1234...",
      botToken: "xoxb-1234...",
      dmPolicy: "pairing"
    }
  }
}
```

これらの設定が完了すれば、各プラットフォームからOpenClawエージェントとの対話が可能になる。

<!-- IMAGE: pairing-sequence.png - DMペアリングとNodeペアリングの認証シーケンス図 -->

---

## 7.7 ペアリングとセキュリティ — 「誰を信頼するか」の制御

OpenClawのチャンネル接続には、2つの「鍵管理」システムがある。誰がメッセージを送信できるかの**DMペアリング（未知の送信者承認）**と、どのデバイスがGatewayにアクセスできるかの**Nodeデバイスペアリング（モバイルアプリ接続）**だ。

### DMペアリング：8文字コードの承認フロー

DMペアリングは、未知の送信者への明示的な承認ステップだ。家の玄関の「インターホン」のようなもので、知らない人が来たら一度確認を求める。

**ペアリングコードの仕様：**
- **8文字大文字**（例：`ABCD1234`）
- **曖昧文字除外**（`0O1I` は含まれない）
- **1時間で期限切れ**
- **チャンネルごと最大3件の保留リクエスト**

**承認手順：**
1. 未知のユーザーがDMを送信
2. OpenClawが8文字コードを生成・送信
3. 管理者が `openclaw pairing list <channel>` で確認
4. `openclaw pairing approve <channel> <CODE>` で承認
5. 以降、そのユーザーからのDMが処理される

**対応チャンネル（20+）：**
BlueBubbles、Discord、Feishu、Google Chat、iMessage、IRC、LINE、Matrix、Mattermost、Microsoft Teams、Nextcloud Talk、Nostr、Signal、Slack、Synology Chat、Telegram、Twitch、WhatsApp、Zalo、Zalo Personal

**状態保存場所：**
- 保留リクエスト：`~/.openclaw/credentials/<channel>-pairing.json`
- 承認済み許可リスト：`~/.openclaw/credentials/<channel>-allowFrom.json`

### Nodeデバイスペアリング：モバイルアプリとの接続

Nodeデバイスペアリングは、iOS/Androidアプリ、macOSアプリ、ヘッドレスNodeがGatewayネットワークに参加する際の承認だ。

**Telegram経由ペアリング（iOS推奨方法）：**

1. **セットアップコード生成**  
   Telegramで `/pair` コマンドを実行

2. **セットアップコード配布**  
   base64エンコードされたJSONをコピー

3. **デバイス側設定**  
   モバイルアプリで 設定 → Gateway → ペースト＆接続

4. **承認処理**  
   Telegramで `/pair pending` → 承認

**セットアップコード内容：**
```json5
{
  "url": "wss://your-gateway.example.com/ws",
  "bootstrapToken": "single-use-bootstrap-token"
}
```

このコードは一回限りのブートストラップトークンを含み、デバイス側で永続的な認証情報に変換される。

**手動承認方法：**
```bash
openclaw devices list                    # 保留中のリクエスト確認
openclaw devices approve <requestId>     # 個別承認
```

**状態保存：**
- `~/.openclaw/devices/pending.json`
- `~/.openclaw/devices/paired.json`

### セキュリティ設計の考慮点

OpenClawのペアリングシステムは、**利便性とセキュリティのバランス**を重視している：

**利便性の配慮：**
- 一度承認すれば、以降の手動承認は不要
- セットアップコードによるワンタイムブートストラップ
- 複数デバイス・複数チャンネルでの同一認証情報共有

**セキュリティの配慮：**
- 8文字コード・1時間TTLによる第三者の介入防止
- チャンネルごと・デバイスタイプごとの独立した許可リスト
- ブートストラップトークンの単発使用による不正アクセス防止

この設計により、「最初の設定だけ少し手間だが、その後は快適」という体験を実現している。

<!-- IMAGE: broadcast-flow.png - 1つのメッセージが複数エージェントに分配されて並行処理される図 -->

---

## 7.8 ブロードキャスト — マルチエージェント応答の実験機能

⚠️ **この節は実験的機能の解説です。基本的なチャンネル接続には不要なので、スキップしてOK**

ブロードキャストは、OpenClaw 2026.1.9で追加された**実験的機能**だ。1つのメッセージに対して複数のエージェントが並行・順次で応答する仕組みで、現在はWhatsAppのみ対応している。

### 「専門家チーム」としてのブロードキャスト

ブロードキャストを「専門家チーム」だと考えるとイメージしやすい。あなたがコードレビューを依頼すると、formatter（整形専門）、security-scanner（セキュリティ専門）、test-coverage（テスト専門）、docs-checker（ドキュメント専門）の4人のエージェントがそれぞれの視点でレビューを返す。

### 設定例

```json5
{
  channels: {
    whatsapp: {
      broadcast: {
        strategy: "parallel",  // または "sequential"
        "120363403215116621@g.us": ["formatter", "security", "test", "docs"],
        "+15555550123": ["support", "logger"]
      }
    }
  }
}
```

この設定では：
- 特定のグループでは4つのエージェントが並行処理
- 特定のDMでは2つのエージェントが並行処理
- `strategy` で並行（`parallel`）か順次（`sequential`）かを選択

### 動作メカニズム

**評価タイミング：** ブロードキャストはチャンネル許可リスト・グループアクティベーション後に評価される。つまり、通常の「参加してよいか？」判定をパスしてから、「複数エージェントで処理するか？」が判定される。

**セッション分離：** 各エージェントは独立したセッションキー・履歴・ワークスペース・ツールアクセス権を持つ。エージェント間での情報漏洩は発生しない。

**失敗処理：** 個別エージェントの失敗は他のエージェントに影響しない。1つのエージェントがエラーを起こしても、他の3つは正常に応答する。

### 実用的な用途例

**1. コードレビューチーム：**
- `formatter` — コードスタイルと整形をチェック
- `security-scanner` — セキュリティ脆弱性を検出
- `test-coverage` — テストケースの妥当性を評価
- `docs-checker` — ドキュメントの整合性を確認

**2. 多言語サポート：**
- `agent-en` — 英語で回答
- `agent-de` — ドイツ語で回答
- `agent-es` — スペイン語で回答

**3. 品質管理：**
- `support-agent` — 顧客からの質問に回答
- `qa-agent` — 回答の品質をチェック・改善提案

**4. タスク自動化：**
- `task-tracker` — タスクをプロジェクト管理ツールに登録
- `time-logger` — 作業時間を記録
- `report-generator` — 週次レポートを生成

### 大規模展開時の注意点

ブロードキャストは実験的機能のため、以下の制限に注意：

**パフォーマンス：**
- 推奨最大5-10エージェント（WhatsAppのAPI率制限に注意）
- 軽量モデル（opus→sonnet）の使用推奨
- 処理時間の増加（parallelでも個別のAPI呼び出しが必要）

**コスト：**
- エージェント数に比例してAPI使用量が増加
- ログファイルサイズの増大

**デバッグ：**
- 複数エージェントの独立ログによるトラブルシューティング複雑化

> 💡 **コラム：ブロードキャストの未来像**  
> 現在はWhatsAppのみだが、将来的にはDiscord、Telegram、Slack等への展開が予定されている。AIエージェントチームの可能性は無限大だ。「企画チーム」「開発チーム」「QAチーム」「営業チーム」のような役割分担で、複雑なプロジェクトを複数の視点から同時に進行させることも可能になるかもしれない。まさに「AIが組織を形成する」未来の先駆けとも言える機能だ。

<!-- IMAGE: plugin-structure.png - チャンネルプラグインのファイル構造とコンポーネント関係図 -->

---

## 7.9 チャンネルプラグイン開発 — 独自プラットフォーム対応

**この節はTypeScript開発者向けです。プラグイン開発に興味がない方はスキップしてOK**

OpenClawの真の価値は、既存のプラットフォーム対応だけでなく、独自のチャンネルプラグインを開発できることにある。社内SNS、自作チャットアプリ、IoTデバイス——任意の通信プラットフォームをOpenClawに接続できる。

### SDKベースの開発フロー

チャンネルプラグイン開発は、OpenClaw SDKの `createChatChannelPlugin` を使用する：

```typescript
import { createChatChannelPlugin, createChannelPluginBase } from '@openclaw/sdk';

export const acmeChatPlugin = createChatChannelPlugin<ResolvedAccount>({
  base: createChannelPluginBase({
    id: "acme-chat",
    setup: { resolveAccount, inspectAccount }
  }),
  
  // セキュリティ（DM制御）
  security: {
    dm: {
      channelKey: "acme-chat",
      resolvePolicy: (account) => account.dmPolicy,
      resolveAllowFrom: (account) => account.allowFrom,
      defaultPolicy: "allowlist"
    }
  },
  
  // ペアリング（テキスト通知）
  pairing: {
    text: {
      idLabel: "Acme Chat username",
      message: "Send this code to verify your identity:",
      notify: async ({ target, code }) => {
        await acmeChatApi.sendDm(target, `Pairing code: ${code}`);
      }
    }
  },
  
  // アウトバウンド（送信）
  outbound: {
    attachedResults: {
      sendText: async (params) => {
        const result = await acmeChatApi.sendMessage(params.to, params.text);
        return { messageId: result.id };
      }
    }
  }
});
```

### 4つの主要コンポーネント

**1. Base（基本構造）：**
- プラグインID・設定解決・アカウント管理
- 最小限の実装でプラグインとして認識される

**2. Security（DM制御）：**
- `dmPolicy`（disabled/allowlist/pairing）の処理
- 許可リスト管理・未承認者の拒否

**3. Pairing（承認フロー）：**
- 8文字コードの生成・配信・期限管理
- プラットフォーム固有の通知方法

**4. Outbound（送信処理）：**
- OpenClaw内部形式→プラットフォームAPI形式の変換
- テキスト・画像・リアクション等の送信

### ファイル構造の標準

```
extensions/acme-chat/
├── package.json              # openclaw.channel メタデータ
├── openclaw.plugin.json      # 設定スキーマ付きマニフェスト  
├── index.ts                  # defineChannelPluginEntry
├── setup-entry.ts            # defineSetupPluginEntry（軽量ロード）
└── src/
    ├── channel.ts            # createChatChannelPlugin実装
    ├── channel.test.ts       # ユニットテスト
    └── client.ts             # プラットフォーム API クライアント
```

**package.json の重要な設定：**
```json5
{
  "name": "@yourcompany/acme-chat",
  "openclaw": {
    "channel": true,
    "entryPoint": "./index.js"
  }
}
```

### インバウンドメッセージハンドリング

プラットフォーム→OpenClawの方向（インバウンド）は、通常WebhookやWebSocketで実装される：

```typescript
// Webhook例（Express.js）
app.post('/acme-webhook', (req, res) => {
  const message = req.body;
  
  // プラットフォーム形式 → OpenClaw内部形式に変換
  const openclawMessage = {
    messageId: message.id,
    text: message.content,
    from: message.sender_id,
    timestamp: new Date(message.created_at),
    attachments: message.media || []
  };
  
  // OpenClaw Gatewayに配信
  gateway.dispatchInboundMessage('acme-chat', openclawMessage);
  
  res.status(200).send('OK');
});
```

このパターンにより、任意のプラットフォームを「OpenClawの言語」に翻訳して接続できる。

### 配布とインストール

完成したプラグインは、npmパッケージとして配布される：

```bash
# 開発者側
npm publish @yourcompany/acme-chat

# ユーザー側
openclaw plugins install @yourcompany/acme-chat
openclaw config set channels.acme-chat.enabled true
```

これにより、OpenClawの公式チャンネルと同等の機能を持つ独自プラットフォーム対応を作成・配布できる。

<!-- IMAGE: troubleshooting-tree.png - 症状から解決策までの診断フローチャート -->

---

## 7.10 トラブルシューティング — 接続できない時の診断と解決

チャンネル接続でトラブルが発生した時は、段階的な診断が重要だ。「症状→原因→解決策」の順で問題を絞り込んでいこう。

> 💬 **こうじの実感：** 最初ハマったのは、Discordで「Message Content Intent」を有効にし忘れたことだった。Botはオンラインになるのに、メッセージに全く反応しない。ログを見ると「content: null」ばかり。Discord Developer Portalでポチッと有効化するだけで解決したけど、3時間悩んだ苦い思い出。トラブル時は、まず権限周りを疑うのが鉄則。

### 診断コマンド階層

トラブルシューティングは、以下の順序で診断コマンドを実行する：

```bash
# レベル1：基本ステータス
openclaw status

# レベル2：Gateway詳細
openclaw gateway status

# レベル3：リアルタイムログ
openclaw logs --follow

# レベル4：総合チェック
openclaw doctor

# レベル5：チャンネル接続確認
openclaw channels status --probe
```

**正常ベースライン：**
- `Runtime: running`
- `RPC probe: ok`
- Channel probe で `connected/ready`

これらのコマンドを上から順に実行し、どの段階で異常が検出されるかで問題を絞り込む。

### チャンネル別の典型的障害パターン

**WhatsApp：**
- **症状：** 接続済みだがDM返信なし
- **原因：** DMペアリング未完了
- **解決策：** `openclaw pairing list whatsapp` → ペアリング承認

- **症状：** グループメッセージ無視
- **原因：** `requireMention`設定・メンションパターン
- **解決策：** メンション方法確認、グループ設定見直し

- **症状：** ランダム切断/再ログインループ
- **原因：** 認証情報の破損・ネットワーク不安定
- **解決策：** `openclaw channels login --channel whatsapp` で再ペアリング

**Telegram：**
- **症状：** `/start`後に応答なし
- **原因：** DMペアリング未承認
- **解決策：** `openclaw pairing approve telegram <CODE>`

- **症状：** Bot オンラインだがグループ沈黙
- **原因：** プライバシーモード有効・メンション不足
- **解決策：** プライバシーモード無効化 または Bot メンション

- **症状：** 送信失敗ネットワークエラー
- **原因：** `api.telegram.org` へのアクセス問題
- **解決策：** DNS/IPv6/プロキシ ルーティング確認

**Discord：**
- **症状：** Bot オンラインだが Guild 返信なし
- **原因：** Message Content Intent未有効化・チャンネル許可不足
- **解決策：** 特権インテント確認、Guild/チャンネル許可設定

- **症状：** グループメッセージ無視
- **原因：** メンションゲーティング
- **解決策：** `requireMention: false` 設定またはBot メンション

- **症状：** DM 応答欠如
- **原因：** DMペアリング未承認
- **解決策：** `openclaw pairing approve discord <CODE>`

**Slack：**
- **症状：** Socket mode接続済みだが応答なし
- **原因：** App Token・Bot Token・スコープ不足
- **解決策：** トークン再確認、必要スコープ追加

- **症状：** チャンネルメッセージ無視
- **原因：** `groupPolicy`設定・チャンネル許可リスト
- **解決策：** グループ設定とチャンネル許可リスト見直し

**iMessage/BlueBubbles：**
- **症状：** インバウンドイベントなし
- **原因：** webhook到達可能性・アプリ権限不足
- **解決策：** macOS Messages自動化のTCC権限再付与

**Signal：**
- **症状：** daemon到達可能だがBot沈黙
- **原因：** `signal-cli` daemon設定・アカウント受信モード
- **解決策：** daemon URL/アカウント設定確認

**Matrix：**
- **症状：** 暗号化ルーム失敗
- **原因：** 暗号化サポート・ルーム同期問題
- **解決策：** 暗号化有効化、ルーム再参加/同期

### ログ解析のポイント

`openclaw logs --follow` の出力で注目すべきポイント：

**正常パターン：**
```
[CHANNEL:discord] Message received from 990665...
[SESSION] Resolved key: agent:main:discord:direct:990665...
[AGENT] Processing message for agent 'main'
[TOOLS] Executing message tool
[CHANNEL:discord] Message sent successfully
```

**異常パターンの典型例：**
```
[CHANNEL:discord] Message ignored: policy violation
→ DMペアリング・許可リスト確認

[CHANNEL:discord] Send failed: Missing permissions
→ Bot権限・チャンネル権限確認

[SESSION] No agent found for peer
→ エージェント設定・ルーティング確認
```

### プリベンティブ（予防的）チェック

定期的に実行すべき健全性チェック：

```bash
# 月次チェック
openclaw doctor

# 週次チェック  
openclaw channels status --probe

# トークン期限チェック（該当チャンネル）
openclaw config get channels.discord.token
openclaw config get channels.slack.botToken
```

これらの診断手順により、チャンネル接続の問題を迅速に特定・解決できる。

---

## 第7章のまとめ — 外部世界との接続を完全制覇

この章では、OpenClawのチャンネル接続システムの全貌を解き明かした。「翻訳者」としてのチャンネル、20+のプラットフォーム対応、プラグインアーキテクチャ、セッションキー連携、グループチャット制御、ペアリングセキュリティ……すべてが論理的で拡張可能な設計に基づいている。

### 重要なポイントの振り返り

**1. チャンネル = 翻訳者**  
プラットフォーム固有のAPIとOpenClaw内部形式を相互変換する役割。これにより、Discord、WhatsApp、Telegramの違いをエージェント側が意識する必要がない。

**2. セッション管理との連携**  
第5章で学んだセッションキーの `:discord:` や `:telegram:` 部分が、チャンネルルーティングの核心。メッセージの送信元→送信先が決定論的に決まる。

**3. プラガブル設計**  
コア機能（message ツール・ルーティング）とプラグイン機能（認証・送信）を分離することで、新しいプラットフォームへの対応が容易。

**4. セキュリティ重視**  
DMペアリング・グループポリシー・デバイスペアリングにより、「誰を信頼するか」を精密に制御。

**5. 実験的機能への対応**  
ブロードキャスト機能のように、将来性のある機能も構造的に収容できる設計。

### 次の章への橋渡し

チャンネル接続により、OpenClawは外部世界と「会話」できるようになった。しかし、会話だけでは物足りない。ファイルを読み、コマンドを実行し、APIを呼び、画像を生成する——エージェントの「手足」となるのがツールとスキルだ。

第8章では、OpenClawのツールシステムの全貌を解き明かす。240+の標準ツール、モデル側からのfunction calling、スキルによる専門タスクの自動化。Discord経由で `/weather` と入力すれば天気予報が、`/image` と入力すれば画像生成が始まる裏側を、次章で丸裸にしよう。

チャンネルで「声」を得たOpenClaw。次は「手足」を得る番だ。