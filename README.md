# reddit-monitor

Cowork (Claude Code の scheduled remote agent) で毎朝 Reddit RSS から新着をまとめるプロジェクト。

## 構成

```
reddit-monitor/
├── PROMPT.md                  ← Cowork が毎朝実行するプロンプト本体
├── config/
│   ├── subreddits.yaml        ← トピックとサブの定義、スコア閾値
│   └── exclude_words.yaml     ← 除外ワード
├── state/
│   └── last_seen.json         ← 各サブの最終処理 post ID（自動更新・commit）
└── reports/
    └── YYYY-MM-DD.md          ← 日次レポート（自動生成・commit）
```

## スケジュール

- 毎朝 8:00 JST に Cowork で `PROMPT.md` を実行
- 9:00 までにレポート完成を目標（初週は実行時間を計測）
- 安定したら 8:15 〜 8:30 に前倒し

## 運用

- サブやスコア閾値を調整したい時は `config/subreddits.yaml` を編集
- 除外ワード追加は `config/exclude_words.yaml`
- ステートをリセットしたい時は `state/last_seen.json` を `{}` に
