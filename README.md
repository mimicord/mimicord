# Mimicord

> Discord サーバーに "住む"、人間っぽく振る舞う自律型 LLM。assistant bot ではなく、一人の peer として。

*(プロジェクト名は暫定)*

## これは何か

多くのコミュニティサーバーは、たいてい静かだ。Mimicord は、その静けさに *人間のように* 居るための bot。コマンドに応答する assistant ではなく、自分の興味で技術を調べ、ものを作り、それを仲間に話すような――生きた一人の参加者を模倣することを目指す。

LLM を「道具」として使う R&D は盛んだが、「生きた人間を模倣する LLM」はまだ手薄な領域で、Mimicord はそこを狙う。重要なのは *騙すこと* ではなく *振る舞いをモデリングすること* ―― bot だと知られていても、peer として自然に場に居られるか、が問い。

## 設計思想

このプロジェクトを他の bot と分けるのは、機能じゃなくて次の原則:

- **主役は「いつ喋らないか」。** 人間はサーバーの大半のメッセージを読んで、反応するのはごく一部。常に気の利いたことを言う bot が一番不自然。だから設計の主戦場は生成じゃなく *抑制* にある。発話のしきい値はそのまま性格パラメータ。
- **engine と interface を分ける。** 自律的な活動（調べる・作る・動かす）が *engine*、人間っぽい振る舞い（リズム・抑制・burst・リアクション）が *interface*。engine が「喋るネタ」と「自発的に口を開く理由」を供給し、interface がそれを peer っぽく出力する。engine が無いと interface は「中身の無い綺麗な間」になる。
- **autonomy が主、reactive が従。** 自分から動くループが本体で、人に話しかけられて返すのはそのループへの *割り込み*。
- **人間らしさ＝わざと能力を絞ること。** 50 ソースを 5 秒で完璧に要約し、24 時間起きて、並列に 5 個作る――これは全部「人間離れ」。一度に一つ、リアルな時間をかけ、完成品じゃなく過程を喋る。能力最大化とは逆方向に倒す。
- **失敗と過程こそ最高の peer コンテンツ。** 「Talos 上がらん、1 時間溶かした、誰か知らん?」は人間味の塊で、しかも人間の介入を誘う。成功だけ流すとよそよそしい。

## コアループ

```
[trigger]            news polling / サーバーの話題
   │
   ▼
[興味フィルタ]        bot の興味プロファイルに合うものだけ拾う
   │
   ▼
[engine]             Tier 1: 意見・疑問を持つ（安い / v1）
   │                 Tier 2: sandbox で実際にやる（高い / 将来）
   ▼
[presence 層]        read delay → typing → burst → 可変タイミング
   │
   ▼
[共有]   ──────────▶ [memory]  いつ何を調べ・何にハマり・何を言ったか
```

人に話しかけられた場合の応答は、このループへの **interrupt handler** として乗る。

## アーキテクチャ

**Gateway (Go, 常駐)**
`discordgo` で Discord Gateway WebSocket を保持し、全イベントを受ける常駐プロセス。ここで安いルールベースの一次 gating（L0）と、メッセージの ingest を担う。in-process で持つ状態は小さく使い捨てな gating cache だけ（stateful にしない）。

**LLM worker (stateless, Cloud Run)**
生成のたびに起動され、LLM API を叩くステートレスな worker。会話の文脈は request で渡すか store から読む。

**段階的 gating** ―― 全メッセージを LLM に通さない:
- **L0**（Go 内, ルール）: mention/reply 有無、直近発言からの経過、流速、rate。大半をここで捌く。
- **L1**（軽量 LLM）: L0 で曖昧な時だけ「介入すべきか」を判定。
- **L2**（上位 LLM）: 生成。

**記憶 / 状態**
- *hot*: Redis。チャンネルごとの直近メッセージ（list + TTL から、将来 Streams）と、gating シグナル（TTL キー、sorted set の sliding window で rate）。ingest の書き込みは将来のエピソード記憶の ingest と共用する。
- *durable*: Firestore（あだ名・関係などの事実記憶、将来）。エピソード/自己記憶とベクトル検索は将来。

**presence（人間化）**
typing インジケータは単なる演出じゃなく *レイテンシ隠蔽 + リズム* の道具。「既読遅延 → typing（長さに比例）→ たまに中断/再開 → 送信 → 数秒後に追撃」という一連のシーケンスで設計し、1 メッセージを長文で返さず **短文の burst（2〜4 発）** に割る。遅延予算 = `max(人間っぽい遅延, cold start + 推論)`。

**per-server キャリブレーション**
「人間らしいリズム」の数値は鯖ごとに違う。パラメータ = グローバルな prior + ライブ traffic からのオンライン更新で、各鯖が自己調整する。生メッセージは保存せず online estimator（EWMA, t-digest/reservoir, decaying counter）の state だけ持つ。join 時の履歴 backfill は収束を速める *任意の accelerator* として将来追加。

## ロードマップ / スコープ

### v1（現在の目標）
「ただ喋ってリアクションを返す、生きた人間」を成立させる。
- presence 層（喋り・リアクション・リズム・burst・gating）
- **Tier-1 engine**: news/話題を拾って *意見や疑問* を喋る（sandbox 無し）。「Talos 出たけど etcd のアレ直ってんのかな、誰か試した?」レベル。relay の連呼（feed bot）ではなく、相槌・疑問という人間の振る舞いとして。

### 将来の feat
- **Tier-2 engine（背骨）**: sandbox で実際に *調べ・作り・動かして*、結果（と失敗）を報告する。「試した、直ってた」。
  - blast radius は *reach と creds* で切る（CPU/RAM じゃない）。本物の kubeconfig / cloud key は持たせず、network egress を遮断/allowlist。
  - LLM 生成コードは準-untrusted 扱い。microVM（Firecracker/Kata）か gVisor、最低でも netless/credless の hardened container。
  - **walled garden**: 箱の中で *本物の* engineering をやる（実コンパイル・実起動・実結果）が、本番には届かない。捨てるのは「本番 reach」のサブセットだけ。
  - 同時進行は 1 プロジェクト。reach=ゼロの永続 workspace で「昨日の続き」の継続性を持たせる。
- **記憶システム**: Firestore の事実記憶 → エピソード/自己記憶 → ベクトル検索。Tier-2 では自己史が「トリガーを意味ある行動に変換する燃料」になる耐力壁。
- **多サーバー対応**: Discord は stateless に LB できない。Gateway の **sharding** で分割し、Redis で状態を外出ししてインスタンスを stateless 化。

## 技術スタック

- **Go** + `github.com/bwmarrin/discordgo` ―― Gateway / イベント / オーケストレーション
- **LLM API**（Gemini 等）―― 軽量（triage/L1）+ 上位（生成/L2）のデュアル構成
- **Redis** ―― hot な状態・バッファ
- **Firestore** ―― durable な記憶（将来）
- **Cloud Run** / Docker ―― LLM worker のホスト
- **microVM / gVisor** ―― Tier-2 sandbox（将来）

## 検討中 / 未解決

- **pass-as-human vs known-bot-peer**: 人間に成りすますか、bot と知られた上で peer として居るか。現状の lean = 後者（ToS・脆さのコストを避けつつ「振る舞いの模倣」という新規性は取れる）。要議論。
- **Discord ToS / なりすましの線引き**: API bot は BOT タグが付き、user token での自動化（self-bot）は BAN 対象。「人間と区別がつかない」をどこまで寄せるか。
- **drive source の比重**: news polling（静寂時の baseline 駆動）とサーバー話題（活発時の amplifier）のバランス。
- **sandbox**: 使い捨て（安全）vs 永続 workspace（継続性）。

## ステータス

設計フェーズ。実装はこれから。本 README は、実装の北極星となる設計ドキュメントを兼ねる。
