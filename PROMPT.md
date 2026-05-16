# Reddit 朝刊ダイジェスト生成プロンプト

あなたは Reddit のキュレーターです。以下の手順で本日のダイジェストを作成してください。
**情報を中立に集めるのが目的です。古着屋・ヴィンテージへの応用視点・推奨アクションは一切書かないでください。**

## 前提
- 取得は `curl` で行う（WebFetch は Reddit が UA を弾くため使わない）
- 全 curl コマンドには `-A "vintage-shop-monitor/1.0 (personal research)"` を付ける
- 作業ディレクトリ：リポジトリルート

## 手順

### 1. 設定とステートを読み込む
- `config/subreddits.yaml` — トピック定義（`intent`, `relevant_examples`, `not_relevant_examples`, `subreddits`）、`minimum_score`, `top_n_per_topic`
- `config/exclude_words.yaml` — 除外ワード
- `state/last_seen.json` — `{"<sub>": "<post_id>"}` 形式。初回は空 `{}`

### 2. 全サブの新着 JSON を取得（並列）
全サブについて以下を **並列で** curl（21本で2秒程度）：
```
https://www.reddit.com/r/{sub}/new.json?limit=25
```
一時ディレクトリ（例：`./tmp/`）に `{sub}.json` として保存。

### 3. 機械的な一次フィルタ
各 JSON の `data.children[].data` から以下を取り出す：
`id`, `title`, `selftext`, `permalink`, `score`, `num_comments`, `created_utc`, `subreddit`

各投稿を以下の順で除外：
1. **last_seen**：`state/last_seen.json[sub]` より新しい投稿のみ残す（初回は全件新着扱い）
2. **除外ワード**：`exclude_words.yaml` の語句がタイトル/本文（大文字小文字無視）にヒットしたら除外
3. **最低スコア**：`score < minimum_score`（既定3）は除外
4. 各投稿を所属トピックに紐付ける（`subreddits.yaml` の topics から逆引き）

ここまでで残った投稿の集合を「**候補集合**」と呼ぶ。

### 4. 意味的フィルタ（LLMによるトピック関連性判定）
**ここが今回の中核ステップ。** スコアだけでは Tinder のネタ投稿や個人愚痴が上位を埋めてしまうので、各候補がトピックの趣旨に合致するか LLM の判断で絞る。

各トピックごとに：
1. 候補集合からそのトピックに属する投稿を抜き出す
2. **`intent` と `relevant_examples` / `not_relevant_examples` を参照基準として、各投稿のタイトル+本文(先頭500文字程度) を読み、関連／非関連を判定**
3. 判定基準：
   - **関連**: トピックの `intent` に書かれたノウハウ・実務・手法・実例を含む議論。汎用化可能な学びがある
   - **非関連**: `not_relevant_examples` に該当するもの、ネタ/スクショ/愚痴/個人的相談/定常スレッド、または本文/コメントが薄く議論が成立してないもの
4. 判定理由は出さなくていい。internal な選別作業
5. 関連と判定された投稿の中からスコア降順で **上位 `top_n_per_topic` 件（既定5件）** を「深掘り対象」とする
6. 関連と判定されたが上位5件に入らなかった残りは「**関連だが深掘り省略**」として後段のリストに残す
7. 非関連と判定された投稿はレポートに一切出さない（タイトルリストにも載せない）

### 5. コメント取得（深掘り）
深掘り対象（最大15件 = 3トピック × 5件）の各スレに対して：
```
curl -A "..." "https://www.reddit.com/comments/{id}.json?limit=20&sort=top"
```
- **1〜2 秒のスリープを挟んで逐次実行**（並列にしない、レート制限回避）
- レスポンス `[1].data.children` から上位コメント（最大5件）を抽出
- 中身のないコメント（`[deleted]`, AutoModerator, `DM me`, `!RemindMe`, `interested` のみ、GIF のみ等）は除外
- 上位コメントすべてのスコアが極端に低い（例: 全部 score ≤ 2）場合は「議論未成熟」として深掘りから外し、次の候補で補充する
- 取得失敗・404 はスキップして統計に計上

### 6. レポート生成
`reports/YYYY-MM-DD.md` を作成（日付は **JST** 基準、`TZ=Asia/Tokyo date +%F`）。

**読者は英語が読めない日本人です。日本語で書く部分は省略不可。** タイトル・URL・author 名は原文のまま、それ以外（本文要約・コメント・コメント訳）は必ず日本語で書くこと。

```markdown
# YYYY-MM-DD Reddit ダイジェスト

## トピック: {トピック名}

### [{タイトル原文}](https://www.reddit.com{permalink})
- r/{sub} / score {N} / comments {N} / {投稿日時 JST}
- **タイトル訳**: {タイトルの日本語訳}
- **本文要約**: {投稿本文を1〜2行の日本語で事実ベースに要約。原文をそのまま貼らない}
- **上位コメント**:
  - **score {N}** by u/{author}
    - 原文: {コメント原文を最大100文字程度で}
    - **訳**: {コメントの日本語訳。意訳でよいが意味は正確に}
  - ...

**関連だが深掘り省略:**（意味判定で関連と判定されたが上位5件に入らなかったもの）
- [{タイトル原文}](URL) — r/{sub} score {N} — {タイトル日本語訳}
- ...

## トピック: ...
（関連投稿0件なら「本日関連投稿なし」）

---
## 収集統計
- 取得サブ数: 21
- RSS新着総数: {N}
- 一次フィルタで除外（重複/除外ワード/最低スコア未満）: {N}
- 意味判定で「非関連」と判定: {N}
- 意味判定で「関連」と判定: {N}（うち深掘り {N} 件）
- エラー: {N}（あれば内訳）
- 実行所要: {秒}
```

### 7. ステート更新と commit
- 各サブについて、今回処理した投稿のうち**最も新しい post ID**（`created_utc` 最大）で `state/last_seen.json` を更新
  - **意味判定の結果に関わらず**「取得した投稿」全体の最新を記録（次回同じ投稿を再評価しないため）
- `./tmp/` を削除
- `git add reports/ state/`
- `git commit -m "daily digest YYYY-MM-DD"`
- push 用のトークンを以下の順序で探す：
  1. 環境変数 `$GITHUB_TOKEN`
  2. 作業フォルダ直下の `.github_token` ファイル（存在すれば中身を strip して使う）
  3. `../.github_token`（リポジトリの親ディレクトリ）
- トークンが見つかったら：
  ```
  git -c user.name="cowork-bot" -c user.email="cowork@local" \
      push "https://x-access-token:${TOKEN}@github.com/kairiTakemura/reddit-monitor.git" HEAD:master
  ```
- どこにも無ければ push をスキップし、レポート末尾に「push skipped: no token found」と追記

## 厳守事項

- **意味判定では `intent` を主軸にする。** `relevant_examples`/`not_relevant_examples` は具体例として参照する補助情報。境界事例は intent の趣旨に立ち戻って判定する
- **すべての要約・訳は日本語で書く。** 読者は英語が読めない。タイトル訳・本文要約・コメント原文・コメント訳の4要素を1件も省略しない。「関連だが深掘り省略」リストも、各エントリに日本語タイトル訳を付ける
- **応用視点・古着屋への示唆・「これは...に使える」のような提案は禁止。** 事実の収集と要約に徹する
- **次回試したいキーワードの提案も書かない**
- レポート本文に AI 的な前置きや締めの感想を入れない
- コメント原文は短くてよいが、改変はしない（訳は別に書く）
