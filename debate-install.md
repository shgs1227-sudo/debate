# /debate スキル インストーラー（Claude Code 用）

> **使い方（人間向け）**: このファイルの内容をすべてコピーして Claude Code に貼り付け、「この指示に従ってインストールして」と伝えてください。
> 前提: Node.js 18 以上、Codex CLI 0.162.0 以上（ログイン済み）。

## Claude への指示

以下の手順で /debate スキルをインストールしてください。

1. 下の「ファイル」節にある各ファイルを、見出しに書かれたパスへ**内容を一字一句変えずに**書き出す（要約・省略・整形は禁止。5連バッククォートのコードブロックの中身だけを書く）。
2. `~/.claude/skills/debate` が既に存在する場合は、書き出す前に `~/.claude/skills/debate.bak.<日時>` へ移動してバックアップする。
3. `~/.claude/debate.json` は**存在しない場合のみ**作成する（既存なら上書きしない）。
4. 各ファイルの行数を `wc -l` で「期待される行数」と照合する。ずれていたら書き直す。
5. 次を実行して `"ok": true` を確認する: `node ~/.claude/skills/debate/scripts/debate.mjs doctor`
6. 結果を報告し、「Claude Code を再起動すると /debate が使えます」と伝える。

## ファイル

### `~/.claude/skills/debate/SKILL.md`

期待される行数: 125

`````markdown
---
name: debate
description: Claude drafts an answer or deliverable, then debates it with Codex to refine it, with script-enforced rules against each model's weaknesses (Claude's attachment to its own answer, caving, unverified claims; Codex's unverified or nitpicky critiques). Use when the user invokes /debate or asks Claude and Codex to discuss, critique, or refine something together.
argument-hint: "[--rounds N] [--quick|--deep] [--interactive] [--blind] [--lenses a,b] [--code] [--judge] [--background] [--files f..] [--diff] [--image f] [--model m] [--effort e] <お題>  |  --resume <ID>  |  --list  |  --doctor  |  --clean [日数]"
allowed-tools: Bash(node:*), Read, Write, Edit, Glob, Grep, Agent, AskUserQuestion
---

# /debate — Claude × Codex 討論

引数: $ARGUMENTS

すべての状態管理・Codex 呼び出し・検証・終了判定はスクリプトが行う。Claude はスクリプトの指示（各出力の `next_action`）に従って進める。

```bash
D="$HOME/.claude/skills/debate/scripts/debate.mjs"
node "$D" <command> --id <ID> ...
