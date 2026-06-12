---
title: "Claude Code のスキル(SKILL.md)を 10 本量産してわかった、壊れない書き方"
emoji: "🛠️"
type: "tech"
topics: ["claudecode", "claude", "ai", "自動化"]
published: false
---

Claude Code には「スキル」という仕組みがあります。`~/.claude/skills/<name>/SKILL.md` にやり方を書いたフォルダを置いておくと、Claude が中身を読んで、ふさわしい場面で勝手に発動してくれる——毎週同じ手順を説明し直す必要がなくなる機能です。

私はこの半年、自分の週次運用(ダイジェスト生成、受信トレイ整理、KPI 記録、定期レポート…)をぜんぶスキルに落として回してきました。10 本書いて運用すると、「動くスキル」と「3 週間後に壊れるスキル」の差がはっきり見えてきます。この記事はその設計パターンのまとめです。最後に実物を 1 本、全文貼ります。

## スキルの最小構造

```
~/.claude/skills/
  folder-tidy/
    SKILL.md        # これだけでも動く
  notion-pusher/
    SKILL.md
    scripts/        # 必要なら補助スクリプト
```

SKILL.md は frontmatter + 本文の Markdown です:

```markdown
---
name: folder-tidy
description: Organize a cluttered macOS folder ... Use when the user says "tidy my desktop", "organize downloads", ...
---

# Folder Tidy (macOS)
## When to use
## What it does
## Steps
## CONFIGURE
## Safety
```

ここから、10 本書いて固まった 4 つのパターンを説明します。

## パターン 1: description がすべて(トリガー設計)

スキルが発動するかどうかは、ほぼ frontmatter の `description` で決まります。Claude は起動時にスキル一覧の description を読み、ユーザーの発話とマッチングして発動を判断するからです。本文がどれだけ丁寧でも、description が曖昧だと一生呼ばれません。

壊れる書き方:

```yaml
description: フォルダを整理するスキル
```

動く書き方:

```yaml
description: Organize a cluttered macOS folder (Desktop, Downloads) by file type
  into subfolders, moving leftovers to Trash rather than hard-deleting.
  Use when the user says "tidy my desktop", "organize downloads",
  "clean up files", or "my folder is a mess".
```

ポイントは 2 つ:

- **何をするか**だけでなく、**ユーザーがどう言ったときに使うか**(発話例)を列挙する。「片付けて」「散らかってる」のような曖昧な言い回しほど書く価値があります
- 紛らわしい近縁スキルがあるなら「〜の場合は使わない」も書く(発動の誤爆は誤発動しないことと同じくらい大事)

## パターン 2: CONFIGURE ブロックでハードコードを隔離する

最初の頃、私はパスやトークン名を本文の手順に直書きしていました。これが 3 週間後に壊れる典型原因です。フォルダを移動した、DB を変えた、というときに、手順のどこを直せばいいか自分でも分からなくなる。

今は全スキルに `## CONFIGURE` セクションを置いて、環境依存の値をそこに集約しています:

```markdown
## CONFIGURE
\```
TARGET = "~/Downloads"
ROUTES = { Images: [jpg,png,gif,heic], Docs: [pdf,docx,xlsx,md],
           Archives: [zip,tar,gz], Installers: [dmg,pkg] }
TRASH_JUNK = true
CONFIRM_BEFORE_MOVE = true
\```
```

本文の手順は「`## CONFIGURE` の TARGET を読め」とだけ書く。こうすると:

- 環境が変わっても直す場所が 1 箇所
- 他人(またはもう 1 台の自分のマシン)に配るとき、CONFIGURE だけ書き換えれば動く
- Claude 自身も「どこが設定で、どこがロジックか」を誤解しなくなる

ちなみに API トークンの類は CONFIGURE にも書きません。macOS なら Keychain に入れて、スキルには「`security find-generic-password -s <service>` で取得しろ」と書きます。SKILL.md は平文で配布・バックアップされるものなので、秘密情報を入れた時点で負けです。

## パターン 3: 安全弁を本文に明文化する(AI に渡す自動化の作法)

スキルは「AI が読む運用手順書」なので、人間向け手順書なら省略する安全側の指示を、明示的に書く必要があります。私が全スキルに入れている安全弁は 3 種類:

**1. 破壊的操作の禁止を明文化する**

```markdown
## Safety
- **Never `rm`.** Junk goes to `~/.Trash` (recoverable). Real files only move.
```

「削除して」と頼んだつもりでも、AI には「ゴミ箱へ移動(復元可能)」を既定にさせる。`rm` は不可逆で、自動化の事故は不可逆操作でしか起きません。

**2. dry-run + confirm を既定にする**

```markdown
## Steps
2. Dry-run first: list what would move where. Ask for confirmation.
```

ファイルを動かす・外部に書き込む類のスキルは、まず「何をするつもりか」を出力させてから実行させる。確認を省きたくなったら CONFIGURE のフラグで外せるようにしておく(既定は安全側)。

**3. 「草案まで、投稿はしない」型の線引き**

SNS 投稿文を作るスキルには `Drafts only. This never posts anywhere` と書いています。生成と公開のあいだに必ず人間を挟む。公開操作そのものを自動化したくなったときも、生成スキルとは別スキルに分けます。

この 3 つを書いておくと、スキルを配布した先(同僚や未来の自分)が中身を読んだだけで「これは実行しても安全だ」と監査できる。**スキルは短く、1 ファイルで、読み切れるサイズに保つ**のもこのためです。

## パターン 4: 定期実行と組み合わせる(そして「成功の捏造」を禁じる)

スキルが本領を発揮するのは、スケジュール実行(cron/launchd や Claude Code の scheduled tasks)と組み合わせたときです。「毎週月曜 7 時に X して報告」が、説明ゼロで回り出します。

ここで 10 本中いちばん学びがあったのが、**失敗時の通知設計**です:

```markdown
## Notes
- **Never notify "success" on failure.** Surface the error instead.
- Test the command manually before enabling the timer.
```

定期ジョブで最悪なのは「失敗しているのに成功通知が来続ける」状態です(沈黙より悪い)。スキルに「失敗したらエラーをそのまま報告しろ、成功を捏造するな」と書いておくだけで、この事故はほぼ消えます。あわせて「0 件だったときに何を送るか」(skip 通知を出すか、黙るか)も CONFIGURE で決めておくと、谷間の週に通知が来なくて不安になる問題もなくなります。

## 実物 1 本、全文公開: folder-tidy

パターンを全部入れた実物がこれです。そのまま `~/.claude/skills/folder-tidy/SKILL.md` に置けば動きます:

```markdown
---
name: folder-tidy
description: Organize a cluttered macOS folder (Desktop, Downloads) by file type into subfolders, moving leftovers to Trash rather than hard-deleting. Use when the user says "tidy my desktop", "organize downloads", "clean up files", or "my folder is a mess".
---

# Folder Tidy (macOS)

## When to use
The user's Desktop or Downloads is cluttered and they want it sorted by type, safely.

## What it does
1. Scans the target folder.
2. Routes files into type subfolders (Images, Docs, Archives, Installers, Media, Other).
3. Moves obvious junk (empty files, completed `.download`, installer `.dmg/.pkg`) to **Trash**, never `rm`.
4. Prints a summary of moved/trashed counts before finalizing.

## Steps
1. Read `## CONFIGURE` for the target folder and routing table.
2. Dry-run first: list what would move where. Ask for confirmation.
3. On confirm, move files; send junk to `~/.Trash` (recoverable for 30 days).
4. Report counts per destination.

## CONFIGURE
\```
TARGET = "~/Downloads"
ROUTES = { Images: [jpg,png,gif,heic], Docs: [pdf,docx,xlsx,md],
           Archives: [zip,tar,gz], Installers: [dmg,pkg], Media: [mp4,mov,mp3] }
TRASH_JUNK = true     # empty files + completed installers go to Trash
CONFIRM_BEFORE_MOVE = true
\```

## Example invocation
"Tidy my Downloads" → dry-run preview, then sorts by type and trashes installers.

## Safety
- **Never `rm`.** Junk goes to `~/.Trash` (recoverable). Real files only move.
- Dry-run + confirm by default.
- Skips files modified in the last 24h (likely in-use).
```

短さに注目してください。スキルは「網羅したドキュメント」ではなく「**判断の足場**」です。長いスキルは Claude のコンテキストを食い、人間の監査可能性も下げます。1 画面に収まらなくなったら、分割のサインだと思っています。

## まとめ

- **description に発話例を書く**(発動はここで決まる)
- **CONFIGURE に環境依存を隔離する**(秘密情報は Keychain へ、SKILL.md には入れない)
- **安全弁を明文化する**(rm 禁止 / dry-run + confirm / 草案のみ)
- **定期実行では成功の捏造を禁じる**(失敗はエラーごと報告)

この 4 つを守るだけで、スキルは「デモで動くもの」から「半年ノーメンテで回る資産」になります。

---

### 宣伝(著者の有料テンプレートです)

この規格で書いた 10 本(weekly-digest / notion-pusher / inbox-triage / folder-tidy / meeting-notes / content-drafter / page-watcher / kpi-logger / research-digest / scheduled-report)を、個人環境の値を抜いたサニタイズ版バンドルとして Gumroad で販売しています。本記事の folder-tidy はそのうちの 1 本です。

- **Claude Skill Bundle: Productivity 10** — $49: https://claudeboost.gumroad.com/l/jkqahd
- 開発者向け 3 パック全部入り **The Complete Automation Studio** — $79: https://claudeboost.gumroad.com/l/studio

無料側では、学術論文検索の MCP サーバ [scholar-mcp](https://github.com/kak4343/scholar-mcp)(npm: `@kak4343/scholar-mcp`)も公開しています。質問はコメント欄へどうぞ。
