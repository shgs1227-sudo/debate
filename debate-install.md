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
```

出力はすべて JSON。`ok: false` のときは `error` と `errors` を読み、**原因を直して同じコマンドを再実行する**。スクリプトの判定を回避しようとしてはいけない。

## 絶対に守ること
- `state.json`、`verify/`、`judge/`、`calls/`、`versions/` を手で編集しない。記録はすべてスクリプト経由で行う。
- 証拠を作り話にしない。引用は実ファイルと照合され、検証結果はスクリプトが実行した記録しか有効にならない。
- 指摘を却下・受け入れする判断は、議論に勝つためではなく、成果物の質のために行う。重要度 高 の却下は自分では確定できない（判定役が決める）。
- 判定役の判定（`judge resolve` の結果）や、ユーザーの判断に従う。支持された指摘は必ず反映する。
- Codex に送る内容は OpenAI に送信される。秘密情報チェックで止まったら、取り除くのが原則。`--allow-secrets` はユーザーの明示的な承認を得た場合だけ付ける。
- 検証コマンド（`verify`）は任意のシェルコマンドを実行する。Codex が提案したコマンドは実行前に中身を確認し、書き込み・削除・ネットワーク送信など副作用のあるものは実行しない。

## 引数の扱い
- `--list` → `node "$D" list`。`--doctor` → `node "$D" doctor`（古い常駐プロセスがあれば、ユーザーに確認してから `doctor --fix`）。`--clean [日数]` → `clean --older-than-days N` で対象を表示し、ユーザーの確認後に `--yes` を付けて再実行。
- `--resume <ID>` → `node "$D" status --id <ID>` で `phase` と `next_action` を確認し、下の手順の該当箇所から再開する。
- それ以外はお題として新しい討論を始める。お題が空ならユーザーに尋ねる。

## 手順

### 0. 準備
1. `node "$D" doctor` を実行。`ok: false` なら内容をユーザーに伝えて止める（モデル非対応や古い常駐プロセスはここで分かる）。
2. `node "$D" init --topic "<お題>" [引数のオプションをそのまま]` → `id`・`context_file`・`artifacts` を控える。
   - 既存ファイルを磨く場合は `--artifact <path>` を付ける（複数可）。コードの変更なら `--code`（Git の変更ファイルが対象、`--test "<cmd>"` でテストを毎ラウンド自動実行）。

### 1. 前提とドラフト（Claude 主体）
1. `context_file` の4項目（依頼・制約・想定する読み手・やらないこと）を埋める。ユーザーの依頼から読み取れないことは推測で埋めず、ユーザーに確認するか「なし」と書く。
2. **自分で十分に考えて**ドラフトを作り、`artifacts` のファイルに書く（新規の文章なら `draft.md`）。
3. `--blind` のときは `node "$D" blind --id <ID>` を実行し、返ってきた Codex の独自解（Claude の案を見ずに作ったもの）と自分の案を比べ、取り入れる点があればドラフトに反映する。初稿は既に固定されているので、後から比較できる。

### 2. ラウンド（Codex の批評）
`node "$D" round --id <ID>`（`--background` 指定時は Bash の run_in_background で実行し、完了通知を待つ）
- `issues_to_decide` が今回回答すべき指摘。`backlog_added`（軽微・範囲外）は回答不要。`invalid_issues` は形式不備で無効になった指摘。
- `questions` があれば: `--interactive` ならユーザーに尋ね、そうでなければ前提から分かるものだけ答える。答えは `node "$D" answer --id <ID> --q Q1 --text "..." --by user|claude` で記録。
- `terminal` が入っていたら、このラウンドの回答を済ませた後で終了処理（手順5）へ進む。

### 3. 回答（Claude の検討と修正）
各指摘を検討する。**鵜呑みにも、反射的な却下もしない。** まず事実を確かめる:
- 動作・事実の指摘には `verification` が付いている。中身を確認したうえで `node "$D" verify --id <ID> --issue C1`（別のコマンドなら `--cmd "..."`）。返る `evidence_ref` を証拠に使える。
- 該当箇所を読み、引用（quote）を証拠にする。

決めたら成果物を修正し、`rounds/rN/response.json` を書く（雛形は `response.template.json`、形式は `schemas/response.json`）:
```json
{
  "decisions": [
    { "id": "C1", "decision": "accept", "rationale": "なぜ正しいか",
      "evidence": [{ "type": "quote", "file": "<成果物のパス>", "line_start": 2, "line_end": 2, "text": "該当箇所の原文" }],
      "applied_at": [] },
    { "id": "C2", "decision": "reject", "reason_type": "incorrect", "rationale": "なぜ当たらないか",
      "evidence": [{ "type": "verify", "id": "V1" }] }
  ],
  "extra_changes": [{ "file": "<パス>", "reason": "指摘以外に変更した理由" }]
}
```
- `decision`: accept / partial / reject。`reason_type`（却下時に必須）: incorrect / out_of_scope / tradeoff / duplicate（`duplicate_of` 必須）/ already_handled。
- 証拠: `quote`（引用。修正前の版を引くときは `"version": "rN"`）/ `verify`（スクリプトの検証記録）/ `url`（外部資料。照合できないので単独では不十分）。
- 修正箇所が指摘の場所と違うときは `applied_at` に実際の場所を書く。成果物以外のファイルを直すときは、**編集前に** `node "$D" add-artifact --id <ID> --path <file>`。
- 指摘に結びつかない変更は `extra_changes` で宣言する（宣言のない変更はエラーになる）。

`node "$D" respond --id <ID>` で検証される。エラーなら直して再実行。スクリプトが課すルール:
- 未回答の指摘があると通らない（T3）。
- 却下には証拠が必要。照合できない証拠での却下は、判定役に回される（W1・G1）。
- 重要度 高 の却下は必ず判定役に回る（W2）。却下が多すぎるとき、同じ指摘を何度も却下したときも回る（W3）。
- 重要度 高 を受け入れるには照合済みの証拠が必要。受け入れた指摘が実際に修正されていないと通らない（W4）。

### 4. 判定（エスカレーションがあった場合）
1. `node "$D" judge run --id <ID>` — Codex 判定役は自動で実行される。
2. `claude_agent_tasks` があれば、各タスクについて **議論の経緯を一切渡さずに** 新しいサブエージェントを起動する（Agent ツール、general-purpose、可能なら並列）。プロンプトは次の内容だけにする:
   > あなたは中立の判定者です。`<packet>` を Read で読み、その指示どおりに判定してください。他のファイルは読まないでください。判定を JSON（case_nonce, winner, severity, confidence, reasoning）で `<output_file>` に Write し、`<submit_command>` を Bash で実行してください。最後に submit の出力をそのまま返してください。
3. 全件そろったら `node "$D" judge resolve --id <ID>`。
   - `must_apply` の指摘は判定役が支持したもの。**必ず修正して反映する**（次の round で確認される）。
   - `held`（判定が割れた）: `--interactive` なら AskUserQuestion でユーザーに選んでもらい `node "$D" decide --id <ID> --issue C1 --decision accept|reject --note "..."` で記録。そうでなければレポートに「要判断」として残る。

続けて `round` に戻る（スクリプトが反映を確認する）。

### 5. 終了処理
1. `node "$D" final-check --id <ID>` — 議論を知らない Codex が最終版を一度だけ見る（W8）。`new_issues` があればユーザーに示し、対応するなら respond / decide で記録してから修正する。
2. `node "$D" report --id <ID>` → `report.md`・`result.json`・`diff.patch` が作られる。

### 6. ユーザーへの出力
```
## 議論の要約
- 結論: <terminal の内容>（N ラウンド）
- Round 1: Codex の主な指摘 → 対応（受け入れ a / 一部 b / 却下 c、判定役 d 件: 結果）
- ...
- 最終確認: <結果>

## 最終版
<成果物（文章ならそのまま、ファイルならパスと変更の要点）>

## 要判断・未解決
<held / final_open / 未回答の質問。なければ省略>
```
最後にレポートのパスを示す。`--background` のときは PushNotification（ToolSearch で読み込む）で完了を知らせる。

## オプション一覧
| オプション | 意味 |
|---|---|
| `--rounds N` | 往復の上限（既定 3） |
| `--quick` / `--deep` | 1往復・推論軽め・判定役と最終確認なし / 5往復・推論深め |
| `--interactive` | 判定が割れた指摘と質問をユーザーに確認しながら進める |
| `--blind` | Codex にも Claude の案を見せずに独自に解かせ、比較する |
| `--lenses a,b` | 観点ごとに Codex の会話を分けて並行批評し、重複を統合する |
| `--code` | Git の変更ファイルを対象にし、差分とテスト結果を毎ラウンド渡す（`--test "<cmd>"`） |
| `--judge` | 上限到達・平行線で残った却下（中以上）も判定役に回す |
| `--background` | 時間のかかる処理を裏で実行し、完了を通知する |
| `--files` / `--diff` / `--image` | 参考ファイル / Git の差分 / 画像を Codex に渡す |
| `--model` / `--effort` | Codex のモデル・推論の深さ（既定は ~/.codex/config.toml に従う） |

既定値は `~/.claude/debate.json`（プロジェクトごとに `.claude/debate.json`）で変えられる。記録は作業フォルダの `.claude/debates/<ID>/` に残る。
`````

### `~/.claude/skills/debate/README.md`

期待される行数: 15

`````markdown
# /debate

Claude × Codex の討論で解答・成果物を洗練する Claude Code スキル。

- `SKILL.md` — Claude 向けの手順書（/debate で読み込まれる）
- `scripts/debate.mjs` — 状態管理・Codex 呼び出し・検証・終了判定を行う本体（依存なし、Node 18+）
- `schemas/` — Codex の返答形式（critique / judge / blind）と Claude の回答形式（response）
- `tests/` — 偽の Codex / Claude CLI を使った回帰テスト

```bash
node --test tests/debate.test.mjs
node scripts/debate.mjs doctor
```

既定値は `~/.claude/debate.json`、プロジェクトごとの上書きは `<project>/.claude/debate.json`。
`````

### `~/.claude/skills/debate/scripts/debate.mjs`

期待される行数: 1783

`````javascript
#!/usr/bin/env node
// Claude × Codex 討論オーケストレーター（/debate スキル用）
//
// 状態管理・Codex 呼び出し・証拠の照合・反映確認・終了判定をすべてこのスクリプトで行い、
// Claude 側の「気分」で議論の流れが変わる余地を減らす。出力はすべて JSON（stdout）。

import fs from 'node:fs';
import path from 'node:path';
import os from 'node:os';
import crypto from 'node:crypto';
import { spawn, spawnSync } from 'node:child_process';
import { fileURLToPath } from 'node:url';

const SCRIPT_PATH = fileURLToPath(import.meta.url);
const SKILL_DIR = path.resolve(path.dirname(SCRIPT_PATH), '..');
const SCHEMA_DIR = path.join(SKILL_DIR, 'schemas');
const HOME = os.homedir();
const INDEX_FILE = process.env.DEBATE_INDEX_FILE || path.join(HOME, '.claude', 'debate-index.jsonl');
const USER_CONFIG = process.env.DEBATE_USER_CONFIG || path.join(HOME, '.claude', 'debate.json');

const SEVERITY_RANK = { low: 0, medium: 1, high: 2 };
const SEVERITY_LABEL = { high: '高', medium: '中', low: '低' };
const CONTEXT_SECTIONS = ['依頼', '制約', '想定する読み手', 'やらないこと'];
const REASON_TYPES = ['incorrect', 'out_of_scope', 'tradeoff', 'duplicate', 'already_handled'];
const REASON_LABEL = {
  incorrect: '指摘が事実と異なる',
  out_of_scope: '依頼の範囲外',
  tradeoff: '取捨選択の結果',
  duplicate: '他の指摘と重複',
  already_handled: '既に対応済み',
};
const STATUS_LABEL = {
  open: '未回答', accepted: '受け入れ', partial: '一部受け入れ', rejected: '却下',
  escalated: '判定待ち', held: '保留（ユーザー判断）', backlog: '後回し', final_open: '最終確認で新規',
};
const EFFORTS = ['none', 'minimal', 'low', 'medium', 'high', 'xhigh'];

const DEFAULTS = {
  rounds: 3,
  effort: null,
  model: null,
  timeoutSec: 900,
  retries: 1,
  logDir: '.claude/debates',
  blockingSeverities: ['high'],
  escalation: { rejectRepeat: 2, rejectRateThreshold: 0.6, minDecisionsForRate: 3 },
  judges: ['claude', 'codex'],
  claudeJudgeMode: 'agent',
  finalCheck: true,
  minCodexVersion: '0.162.0',
  codexBin: 'codex',
  claudeBin: null,
  maxInlineBytes: 200000,
  verifyTimeoutSec: 300,
  verifyOutputBytes: 20000,
  testCommands: [],
  applyTolerance: 1,
  secretPatterns: [],
  secretAllow: [],
  presets: {
    quick: { rounds: 1, effort: 'low', judges: [], finalCheck: false },
    deep: { rounds: 5, effort: 'xhigh' },
  },
};

const BUILTIN_SECRET_PATTERNS = [
  ['aws-access-key', 'AKIA[0-9A-Z]{16}', 'g'],
  ['api-key', 'sk-(?:ant-|proj-)?[A-Za-z0-9_-]{20,}', 'g'],
  ['github-token', 'gh[pousr]_[A-Za-z0-9]{36,}', 'g'],
  ['slack-token', 'xox[baprs]-[A-Za-z0-9-]{10,}', 'g'],
  ['google-api-key', 'AIza[0-9A-Za-z_-]{35}', 'g'],
  ['private-key', '-----BEGIN [A-Z ]*PRIVATE KEY-----', 'g'],
  ['jwt', 'eyJ[A-Za-z0-9_-]{10,}\\.[A-Za-z0-9_-]{10,}\\.[A-Za-z0-9_-]{10,}', 'g'],
  ['credential-assignment', '(?:password|passwd|secret|api[_-]?key|access[_-]?token)\\s*[:=]\\s*["\'][^"\'\\s]{8,}["\']', 'gi'],
];
const SECRET_FILE_RE = /(^|\/)(\.env(\..+)?|[^/]*\.pem|[^/]*\.key|id_rsa[^/]*|id_ed25519[^/]*|[^/]*\.p12|[^/]*\.pfx|credentials(\.[^/]+)?|\.npmrc|\.pypirc)$/i;
const SECRET_FILE_OK_RE = /\.env\.(example|sample|template)$/i;
const MODEL_ERR_RE = /model is not supported|model_not_found|does not exist or you do not have access/i;

// ───────────────────────── 汎用ヘルパー ─────────────────────────

class UserError extends Error {
  constructor(msg, extra = {}) { super(msg); this.extra = extra; }
}
const fail = (msg, extra) => { throw new UserError(msg, extra); };
const out = (obj) => process.stdout.write(JSON.stringify(obj, null, 2) + '\n');
const now = () => new Date().toISOString();
const sha256 = (s) => crypto.createHash('sha256').update(s).digest('hex');
const nonEmpty = (s) => typeof s === 'string' && s.trim().length > 0;
const rank = (sev) => SEVERITY_RANK[sev] ?? 0;
const maxSeverity = (list) => list.reduce((a, b) => (rank(b) > rank(a) ? b : a), 'low');
const readJSON = (f) => JSON.parse(fs.readFileSync(f, 'utf8'));
const exists = (f) => fs.existsSync(f);

function writeJSON(f, obj) {
  fs.mkdirSync(path.dirname(f), { recursive: true });
  const tmp = `${f}.tmp-${process.pid}`;
  fs.writeFileSync(tmp, JSON.stringify(obj, null, 2));
  fs.renameSync(tmp, f);
}
function writeText(f, s) {
  fs.mkdirSync(path.dirname(f), { recursive: true });
  fs.writeFileSync(f, s);
}
function truncate(s, n) {
  s = String(s ?? '');
  return s.length > n ? `${s.slice(0, n)}\n…（以下 ${s.length - n} 文字省略）` : s;
}
function numbered(text) {
  return text.split('\n').map((l, i) => `${String(i + 1).padStart(5)}| ${l}`).join('\n');
}
function compareVersions(a, b) {
  const pa = String(a).split(/[.-]/).map((x) => parseInt(x, 10) || 0);
  const pb = String(b).split(/[.-]/).map((x) => parseInt(x, 10) || 0);
  for (let i = 0; i < Math.max(pa.length, pb.length); i++) {
    if ((pa[i] ?? 0) !== (pb[i] ?? 0)) return (pa[i] ?? 0) - (pb[i] ?? 0);
  }
  return 0;
}
const isObj = (v) => v && typeof v === 'object' && !Array.isArray(v);
function merge(a, b) {
  const r = { ...a };
  for (const [k, v] of Object.entries(b || {})) r[k] = isObj(v) && isObj(a[k]) ? merge(a[k], v) : v;
  return r;
}
function normText(s) {
  return String(s ?? '').replace(/\s+/g, ' ').trim();
}

// ───────────────────────── 引数 ─────────────────────────

const BOOL_FLAGS = new Set([
  'quick', 'deep', 'interactive', 'blind', 'code', 'judge', 'background', 'diff',
  'fix', 'dry-run', 'yes', 'allow-secrets', 'json',
]);
const MULTI_FLAGS = new Set(['files', 'image', 'add-dir', 'artifact', 'test']);
const SPLIT_FLAGS = new Set(['files', 'image', 'add-dir', 'artifact']);

function parseArgs(argv) {
  const o = { _: [] };
  for (let i = 0; i < argv.length; i++) {
    const a = argv[i];
    if (!a.startsWith('--')) { o._.push(a); continue; }
    const k = a.slice(2);
    if (BOOL_FLAGS.has(k)) { o[k] = true; continue; }
    const v = argv[++i];
    if (v === undefined) fail(`${a} に値がありません`);
    if (MULTI_FLAGS.has(k)) {
      const vals = SPLIT_FLAGS.has(k) ? v.split(',').map((s) => s.trim()).filter(Boolean) : [v];
      (o[k] ??= []).push(...vals);
    } else o[k] = v;
  }
  return o;
}

// ───────────────────────── 設定・状態 ─────────────────────────

function loadConfig(cwd) {
  let c = structuredClone(DEFAULTS);
  for (const f of [USER_CONFIG, path.join(cwd, '.claude', 'debate.json')]) {
    if (!exists(f)) continue;
    try { c = merge(c, readJSON(f)); } catch (e) { fail(`設定ファイルを読めません: ${f}: ${e.message}`); }
  }
  return applyEnvOverrides(c);
}
function applyEnvOverrides(s) {
  if (process.env.DEBATE_CODEX_BIN) s.codexBin = process.env.DEBATE_CODEX_BIN;
  if (process.env.DEBATE_CLAUDE_BIN) s.claudeBin = process.env.DEBATE_CLAUDE_BIN;
  return s;
}

function indexRead() {
  if (!exists(INDEX_FILE)) return [];
  const byId = new Map();
  for (const line of fs.readFileSync(INDEX_FILE, 'utf8').split('\n')) {
    if (!line.trim()) continue;
    try { const e = JSON.parse(line); byId.set(e.id, e); } catch { /* 壊れた行は無視 */ }
  }
  return [...byId.values()];
}
function indexAdd(entry) {
  fs.mkdirSync(path.dirname(INDEX_FILE), { recursive: true });
  fs.appendFileSync(INDEX_FILE, JSON.stringify(entry) + '\n');
}

function loadDebate(opts) {
  let id = opts.id;
  const idx = indexRead();
  if (!id || id === 'last') {
    const cwd = process.cwd();
    const mine = idx.filter((e) => e.cwd === cwd);
    if (!mine.length) fail('このフォルダの討論が見つかりません。--id を指定してください');
    id = mine[mine.length - 1].id;
  }
  const entry = idx.find((e) => e.id === id);
  const dir = entry?.dir ?? path.resolve(process.cwd(), DEFAULTS.logDir, id);
  const sf = path.join(dir, 'state.json');
  if (!exists(sf)) fail(`討論 ${id} が見つかりません（${dir}）`);
  const state = readJSON(sf);
  applyEnvOverrides(state.settings);
  return { dir, state };
}

const NEXT_ACTION = {
  drafting: 'context.md の4項目と成果物（ドラフト）を書き、`round` を実行する（--blind なら先に `blind`）',
  awaiting_response: '未回答の指摘に回答ファイル（rounds/rN/response.json）を書き、修正を反映してから `respond` を実行する',
  awaiting_judge: '`judge run` で判定を依頼し、Claude 判定役の提出後に `judge resolve` を実行する',
  awaiting_user: '保留中の指摘をユーザーに確認し、`decide` で記録する',
  awaiting_codex: '次の `round` を実行する（判定で支持された指摘があれば、先に反映する）',
  finalizing: '判定で支持された指摘があれば反映し、`final-check` を実行する',
  final_review: '最終確認で見つかった指摘をユーザーに示し、対応する場合は respond / decide を行ってから `report` を実行する',
  reporting: '`report` を実行する',
  done: '完了。report.md を確認する',
};

function refreshPhase(state) {
  const all = Object.values(state.issues);
  let phase;
  if (state.round === 0) phase = 'drafting';
  else if (all.some((i) => i.status === 'open')) phase = 'awaiting_response';
  else if (all.some((i) => i.status === 'escalated')) phase = 'awaiting_judge';
  else if (state.settings.interactive && all.some((i) => i.status === 'held')) phase = 'awaiting_user';
  else if (!state.terminal) phase = 'awaiting_codex';
  else if (!state.final_check) phase = 'finalizing';
  else if (all.some((i) => i.status === 'final_open')) phase = 'final_review';
  else phase = state.reported_hash && state.reported_hash === artifactsHash(state) ? 'done' : 'reporting';
  state.phase = phase;
  state.next_action = NEXT_ACTION[phase];
  return phase;
}

function saveState(ctx) {
  ctx.state.updated_at = now();
  refreshPhase(ctx.state);
  writeJSON(path.join(ctx.dir, 'state.json'), ctx.state);
}
function logLine(ctx, text) {
  fs.appendFileSync(path.join(ctx.dir, 'log.md'), `- ${now()} ${text}\n`);
}

// ───────────────────────── 成果物・スナップショット・差分 ─────────────────────────

function artifactList(state) {
  return state.artifacts.map((rel) => ({ rel, abs: path.resolve(state.cwd, rel) }));
}
function readIfExists(f) {
  return exists(f) ? fs.readFileSync(f, 'utf8') : null;
}
function artifactsHash(state) {
  const h = crypto.createHash('sha256');
  for (const a of artifactList(state)) h.update(`${a.rel}\0${readIfExists(a.abs) ?? '<missing>'}\0`);
  return h.digest('hex');
}
function snapshot(ctx, label) {
  const { dir, state } = ctx;
  const vdir = path.join(dir, 'versions', label);
  fs.rmSync(vdir, { recursive: true, force: true });
  fs.mkdirSync(vdir, { recursive: true });
  for (const a of artifactList(state)) {
    if (!exists(a.abs)) continue;
    const dst = path.join(vdir, a.rel);
    fs.mkdirSync(path.dirname(dst), { recursive: true });
    fs.copyFileSync(a.abs, dst);
  }
  const hash = artifactsHash(state);
  state.versions[label] = { hash, at: now() };
  return hash;
}
function toRel(state, p) {
  const abs = path.resolve(state.cwd, p);
  const rel = path.relative(state.cwd, abs);
  if (rel.startsWith('..') || path.isAbsolute(rel)) fail(`作業フォルダの外のファイルは扱えません: ${p}`);
  return rel;
}
function normalizeFile(state, f) {
  if (!nonEmpty(f)) return null;
  let p = f.trim().replace(/^\.\//, '');
  if (path.isAbsolute(p)) p = path.relative(state.cwd, p);
  if (state.artifacts.includes(p)) return p;
  const hit = state.artifacts.find((a) => a.endsWith(`/${p}`) || p.endsWith(`/${a}`) || path.basename(a) === p);
  return hit || p;
}

// diff -U0 の結果から「古い版の何行目が変わったか」を取り出す
function changedRanges(oldAbs, newAbs) {
  const o = exists(oldAbs), n = exists(newAbs);
  if (!o && !n) return [];
  if (!o || !n) return [{ start: 0, end: Infinity }];
  const r = spawnSync('diff', ['-U0', oldAbs, newAbs], { encoding: 'utf8', maxBuffer: 64 << 20 });
  const ranges = [];
  for (const m of (r.stdout || '').matchAll(/^@@ -(\d+)(?:,(\d+))? \+(\d+)(?:,(\d+))? @@/gm)) {
    const s = +m[1];
    const cnt = m[2] === undefined ? 1 : +m[2];
    ranges.push(cnt === 0 ? { start: s, end: s + 1 } : { start: s, end: s + cnt - 1 });
  }
  return ranges;
}
function cycleChanges(ctx, label = `r${ctx.state.round}`) {
  const res = [];
  for (const a of artifactList(ctx.state)) {
    const ranges = changedRanges(path.join(ctx.dir, 'versions', label, a.rel), a.abs);
    if (ranges.length) res.push({ rel: a.rel, ranges });
  }
  return res;
}
function diffVersions(ctx, fromLabel, toLabel) {
  const a = path.join(ctx.dir, 'versions', fromLabel);
  const b = path.join(ctx.dir, 'versions', toLabel);
  if (!exists(a) || !exists(b)) return '';
  const r = spawnSync('diff', ['-ruN', fromLabel, toLabel], { cwd: path.join(ctx.dir, 'versions'), encoding: 'utf8', maxBuffer: 64 << 20 });
  return r.stdout || '';
}
function overlaps(range, target, tol) {
  if (target.line_start == null) return true;
  const ts = target.line_start - tol;
  const te = (target.line_end ?? target.line_start) + tol;
  return range.start <= te && range.end >= ts;
}

// W4・G4: 受け入れた指摘が実際に反映されたか、指摘に結びつかない変更がないか
function applyCheck(ctx) {
  const { state } = ctx;
  const tol = state.settings.applyTolerance;
  const changes = cycleChanges(ctx);
  const cycleIssues = Object.values(state.issues).filter((i) => i.accepted_round === state.round && ['accepted', 'partial'].includes(i.status));
  const declared = new Set(state.cycle_extra.filter((x) => x.round === state.round).map((x) => x.file));
  const errors = [];
  const unattributed = [];
  for (const i of cycleIssues) {
    const targets = i.apply_targets?.length ? i.apply_targets : [i.location];
    let ok = false;
    for (const t of targets) {
      if (!t?.file) continue;
      if (!state.artifacts.includes(t.file)) {
        errors.push(`${i.id}: 反映先 ${t.file} が成果物に含まれていません。編集前に \`add-artifact --path ${t.file}\` を実行するか、applied_at で実際の反映先を指定してください`);
        ok = true; // 二重にエラーを出さない
        continue;
      }
      const ch = changes.find((c) => c.rel === t.file);
      if (ch && ch.ranges.some((rg) => overlaps(rg, t, tol))) ok = true;
    }
    if (!ok) errors.push(`${i.id}: 受け入れた指摘が成果物に反映されていません（${targets.map(fmtLoc).join(', ')} に変更がない）。修正するか、実際の反映箇所を applied_at で指定してください（W4）`);
    else i.applied = true;
  }
  for (const c of changes) {
    if (declared.has(c.rel)) continue;
    for (const rg of c.ranges) {
      const covered = cycleIssues.some((i) => (i.apply_targets?.length ? i.apply_targets : [i.location]).some((t) => t?.file === c.rel && overlaps(rg, t, tol)));
      if (!covered) unattributed.push(`${c.rel}:${rg.start}${rg.end !== rg.start ? `-${rg.end === Infinity ? '末尾' : rg.end}` : ''}`);
    }
  }
  if (unattributed.length) {
    errors.push(`どの指摘にも結びつかない変更があります（G4）: ${unattributed.join(', ')}。元に戻すか、extra_changes / \`declare-change\` で理由を宣言してください`);
  }
  return { errors, unattributed };
}
function fmtLoc(l) {
  if (!l?.file) return '（場所不明）';
  if (l.line_start == null) return l.file;
  return `${l.file}:${l.line_start}${l.line_end && l.line_end !== l.line_start ? `-${l.line_end}` : ''}`;
}

// ───────────────────────── 前提ファイル（W9） ─────────────────────────

function contextTemplate(topic) {
  return `# 前提\n\n<!-- 4つの項目をすべて埋める。該当しない場合は「なし」と書く。Codex にはこのファイルがそのまま渡る -->\n\n## 依頼\n${topic ? `${topic}\n` : ''}\n## 制約\n\n## 想定する読み手\n\n## やらないこと\n`;
}
function parseContext(text) {
  const sections = {};
  let cur = null;
  for (const line of text.replace(/<!--[\s\S]*?-->/g, '').split('\n')) {
    const m = line.match(/^##\s+(.+?)\s*$/);
    if (m) { cur = m[1]; sections[cur] = ''; continue; }
    if (cur) sections[cur] += `${line}\n`;
  }
  return sections;
}
function checkContext(ctx) {
  const f = path.join(ctx.dir, 'context.md');
  const text = readIfExists(f) ?? '';
  const sec = parseContext(text);
  const missing = CONTEXT_SECTIONS.filter((k) => !nonEmpty(sec[k]));
  if (missing.length) fail(`前提ファイル（context.md）の項目が空です（W9）: ${missing.join('、')}。該当しない場合は「なし」と書いてください`, { file: f });
  return text;
}
function checkArtifacts(ctx) {
  const empty = artifactList(ctx.state).filter((a) => !nonEmpty(readIfExists(a.abs) ?? ''));
  if (empty.length === ctx.state.artifacts.length) fail(`成果物が空です: ${empty.map((a) => a.rel).join(', ')}`);
}

// ───────────────────────── 秘密情報チェック（I2） ─────────────────────────

function maskSecret(s) {
  return s.length <= 6 ? '***' : `${s.slice(0, 4)}…${'*'.repeat(Math.min(8, s.length - 4))}`;
}
function scanSecrets(state, texts, files) {
  const s = state.settings;
  const pats = [...BUILTIN_SECRET_PATTERNS, ...(s.secretPatterns || []).map((p, i) => [`custom-${i + 1}`, p, 'g'])];
  const allow = (s.secretAllow || []).map((a) => new RegExp(a));
  const findings = new Map();
  for (const f of files) {
    if (SECRET_FILE_RE.test(f) && !SECRET_FILE_OK_RE.test(f)) findings.set(`file:${f}`, { where: f, kind: 'secret-file', sample: path.basename(f) });
  }
  for (const { label, text } of texts) {
    String(text ?? '').split('\n').forEach((ln, idx) => {
      for (const [name, src, flags] of pats) {
        for (const m of ln.matchAll(new RegExp(src, flags))) {
          if (allow.some((a) => a.test(m[0]))) continue;
          findings.set(`${label}:${idx + 1}:${m[0]}`, { where: `${label}:${idx + 1}`, kind: name, sample: maskSecret(m[0]) });
        }
      }
    });
  }
  return [...findings.values()];
}
function guardSecrets(ctx, opts, prompt, label) {
  const { state } = ctx;
  const files = [...state.artifacts, ...(state.settings.refFiles || []), ...(state.settings.images || [])];
  const texts = [{ label: `送信内容(${label})`, text: prompt }];
  for (const f of [...state.artifacts, ...(state.settings.refFiles || [])]) {
    texts.push({ label: f, text: readIfExists(path.resolve(state.cwd, f)) ?? '' });
  }
  const findings = scanSecrets(state, texts, files);
  if (!findings.length) return;
  if (opts['allow-secrets']) {
    state.secret_overrides = [...(state.secret_overrides || []), { at: now(), label, findings }];
    logLine(ctx, `秘密情報チェックをユーザー承認で通過（${label}、${findings.length} 件）`);
    return;
  }
  fail('Codex に送る内容に秘密情報らしきものが含まれているため、送信を止めました（I2）。取り除くか、ユーザーの承認を得てから --allow-secrets を付けて再実行してください', { findings });
}

// ───────────────────────── 外部プロセス ─────────────────────────

function runProc(cmd, args, { cwd, input, timeoutMs } = {}) {
  return new Promise((resolve) => {
    const started = Date.now();
    let settled = false;
    const done = (r) => { if (!settled) { settled = true; resolve({ ...r, duration: Date.now() - started }); } };
    let child;
    try {
      child = spawn(cmd, args, { cwd, env: process.env, stdio: ['pipe', 'pipe', 'pipe'] });
    } catch (e) {
      done({ code: -1, stdout: '', stderr: String(e), timedOut: false });
      return;
    }
    let stdout = '';
    let stderr = '';
    let timedOut = false;
    child.stdout.on('data', (d) => { stdout += d; });
    child.stderr.on('data', (d) => { stderr += d; });
    const timer = timeoutMs
      ? setTimeout(() => {
        timedOut = true;
        child.kill('SIGTERM');
        setTimeout(() => child.kill('SIGKILL'), 5000).unref();
      }, timeoutMs)
      : null;
    child.on('error', (e) => { if (timer) clearTimeout(timer); done({ code: -1, stdout, stderr: `${stderr}${e}`, timedOut }); });
    child.on('close', (code) => { if (timer) clearTimeout(timer); done({ code: code ?? -1, stdout, stderr, timedOut }); });
    child.stdin.on('error', () => {});
    child.stdin.end(input ?? '');
  });
}

function parseCodexEvents(text) {
  const ev = { threadId: null, usage: { input: 0, cached: 0, output: 0, reasoning: 0 }, errors: [] };
  for (const line of String(text).split('\n')) {
    if (!line.startsWith('{')) continue;
    let e;
    try { e = JSON.parse(line); } catch { continue; }
    if (e.type === 'thread.started' && e.thread_id) ev.threadId = e.thread_id;
    if (e.type === 'turn.completed' && e.usage) {
      ev.usage.input += e.usage.input_tokens || 0;
      ev.usage.cached += e.usage.cached_input_tokens || 0;
      ev.usage.output += e.usage.output_tokens || 0;
      ev.usage.reasoning += e.usage.reasoning_output_tokens || 0;
    }
    if (e.type === 'error') ev.errors.push(e.message || JSON.stringify(e));
    if (e.type === 'turn.failed') ev.errors.push(e.error?.message || JSON.stringify(e));
  }
  return ev;
}

// Codex を読み取り専用で呼ぶ。threadId があればその会話を続ける（A2）。依頼文は標準入力で渡す（A1）。
async function callCodex(ctx, { label, prompt, schema, threadId = null, ephemeral = false, images = [], round = null }) {
  const { dir, state } = ctx;
  const s = state.settings;
  const callDir = path.join(dir, 'calls');
  writeText(path.join(callDir, `${label}.prompt.md`), prompt);
  const outFile = path.join(callDir, `${label}.out.json`);
  const common = ['--json', '--skip-git-repo-check', '--output-schema', path.join(SCHEMA_DIR, schema), '-o', outFile];
  if (s.model) common.push('-m', s.model);
  if (s.effort) common.push('-c', `model_reasoning_effort="${s.effort}"`);
  const maxAttempts = 1 + Math.max(0, s.retries ?? 0);
  let last = null;
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    fs.rmSync(outFile, { force: true });
    let args;
    if (threadId) {
      args = ['exec', 'resume', threadId, ...common, '-c', 'sandbox_mode="read-only"', '-'];
    } else {
      args = ['exec', ...common, '-s', 'read-only', '-C', state.cwd];
      if (ephemeral) args.push('--ephemeral');
      for (const img of images) args.push('-i', img);
      for (const d of s.addDirs || []) args.push('--add-dir', d);
      args.push('-');
    }
    const r = await runProc(s.codexBin, args, { cwd: state.cwd, input: prompt, timeoutMs: s.timeoutSec * 1000 });
    writeText(path.join(callDir, `${label}.events.jsonl`), r.stdout);
    if (r.stderr) writeText(path.join(callDir, `${label}.stderr.txt`), r.stderr);
    const ev = parseCodexEvents(r.stdout);
    const errText = `${ev.errors.join('\n')}\n${r.stderr}`;
    if (MODEL_ERR_RE.test(errText)) {
      state.metrics.push({ label, agent: 'codex', round, ok: false, attempts: attempt, duration_ms: r.duration, at: now() });
      fail('Codex がモデルを受け付けませんでした。Codex CLI が古いか、更新前の常駐プロセスが残っている可能性があります（A3）。`doctor` を実行してください', {
        detail: truncate(errText.trim(), 600),
        stale_processes: findStaleCodexProcesses(s.codexBin),
      });
    }
    last = { r, ev, attempt };
    if (r.code === 0 && !r.timedOut && exists(outFile)) {
      try {
        const data = JSON.parse(fs.readFileSync(outFile, 'utf8'));
        state.metrics.push({ label, agent: 'codex', round, ok: true, attempts: attempt, duration_ms: r.duration, usage: ev.usage, at: now() });
        return { data, threadId: ev.threadId || threadId, usage: ev.usage, duration: r.duration, attempts: attempt };
      } catch (e) {
        last.parseError = e.message;
      }
    }
  }
  state.metrics.push({ label, agent: 'codex', round, ok: false, attempts: maxAttempts, duration_ms: last?.r.duration, at: now() });
  fail(`Codex の呼び出しに失敗しました（${label}、${maxAttempts} 回試行）（A4）。状態は保存済みなので、同じコマンドを再実行すれば続きから試せます`, {
    timed_out: last?.r.timedOut,
    exit_code: last?.r.code,
    parse_error: last?.parseError,
    errors: last?.ev.errors,
    stderr_tail: truncate((last?.r.stderr || '').slice(-2000), 2000),
  });
}

function findClaudeBin(s) {
  if (s.claudeBin) return s.claudeBin;
  const base = path.join(HOME, 'Library', 'Application Support', 'Claude', 'claude-code');
  try {
    const versions = fs.readdirSync(base).filter((v) => /^\d/.test(v)).sort(compareVersions).reverse();
    for (const v of versions) {
      for (const sub of fs.readdirSync(path.join(base, v))) {
        const bin = path.join(base, v, sub, 'claude.app', 'Contents', 'MacOS', 'claude');
        if (exists(bin)) return bin;
      }
    }
  } catch { /* なければ PATH の claude */ }
  return 'claude';
}

async function callClaudeCli(ctx, { label, prompt, schema, round = null }) {
  const { dir, state } = ctx;
  const s = state.settings;
  const bin = findClaudeBin(s);
  writeText(path.join(dir, 'calls', `${label}.prompt.md`), prompt);
  const schemaText = fs.readFileSync(path.join(SCHEMA_DIR, schema), 'utf8');
  const args = ['-p', '--output-format', 'json', '--json-schema', schemaText, '--tools', '', '--no-session-persistence', '--strict-mcp-config'];
  const r = await runProc(bin, args, { cwd: state.cwd, input: prompt, timeoutMs: s.timeoutSec * 1000 });
  writeText(path.join(dir, 'calls', `${label}.out.json`), r.stdout);
  let res;
  try { res = JSON.parse(r.stdout); } catch { res = null; }
  const ok = res && !res.is_error && res.structured_output;
  state.metrics.push({
    label, agent: 'claude', round, ok: !!ok, attempts: 1, duration_ms: r.duration, at: now(),
    usage: res?.usage ? { input: res.usage.input_tokens || 0, output: res.usage.output_tokens || 0 } : undefined,
    cost_usd: res?.total_cost_usd,
  });
  if (!ok) {
    fail('Claude CLI での判定に失敗しました。設定 claudeJudgeMode を "agent" にするとサブエージェント方式で判定できます', {
      result: res?.result, stderr_tail: truncate(r.stderr.slice(-1000), 1000),
    });
  }
  return res.structured_output;
}

function findStaleCodexProcesses(bin) {
  const w = spawnSync('/bin/sh', ['-c', `command -v "${bin}"`], { encoding: 'utf8' });
  const p = (w.stdout || '').trim();
  if (!p) return [];
  let installedAt = 0;
  try {
    const real = fs.realpathSync(p);
    const st = fs.statSync(real);
    const lst = fs.lstatSync(p);
    installedAt = Math.max(st.mtimeMs, st.ctimeMs, lst.mtimeMs, lst.ctimeMs);
  } catch { return []; }
  const ps = spawnSync('ps', ['-axo', 'pid=,lstart=,command='], { encoding: 'utf8' });
  const res = [];
  for (const line of (ps.stdout || '').split('\n')) {
    const m = line.match(/^\s*(\d+)\s+(\w{3}\s+\w{3}\s+\d+\s+[\d:]+\s+\d{4})\s+(.*)$/);
    if (!m) continue;
    const cmd = m[3];
    if (!/app-server-broker\.mjs/.test(cmd) && !/^codex app-server(\s|$)/.test(cmd)) continue;
    const started = Date.parse(m[2]);
    if (Number.isFinite(started) && started < installedAt) res.push({ pid: +m[1], started: m[2], command: truncate(cmd, 160) });
  }
  return res;
}

// ───────────────────────── 依頼文の組み立て ─────────────────────────

const FILE_BEGIN = (p) => `<<<FILE ${p}>>>`;
const FILE_END = '<<<END FILE>>>';

function sectionFiles(state, rels, heading, { withNumbers = true } = {}) {
  if (!rels.length) return '';
  const items = rels.map((rel) => ({ rel, text: readIfExists(path.resolve(state.cwd, rel)) }));
  const total = items.reduce((n, x) => n + (x.text?.length ?? 0), 0);
  let body;
  if (total <= state.settings.maxInlineBytes) {
    body = items.map((x) => `${FILE_BEGIN(x.rel)}\n${x.text == null ? '（ファイルがありません）' : withNumbers ? numbered(x.text) : x.text}\n${FILE_END}`).join('\n\n');
  } else {
    body = `サイズが大きいため本文は省略します。次のファイルを直接読んでください（作業フォルダからの相対パス。行番号は実ファイル基準）:\n${items.map((x) => `- ${x.rel}`).join('\n')}`;
  }
  return `# ${heading}\n${body}\n`;
}

function gitDiffText(state) {
  const r = spawnSync('git', ['-C', state.cwd, 'diff', 'HEAD'], { encoding: 'utf8', maxBuffer: 64 << 20 });
  return r.status === 0 ? r.stdout : '';
}

const RULES = `# 出力のルール（スクリプトが機械的にチェックします）
- 出力は指定された JSON スキーマに従ってください。ファイルの編集はできません（読み取り専用）。必要ならファイルを読んだり、読み取りだけのコマンドを実行して確かめてください。
- issues の各指摘:
  - severity: high（結論や動作を誤らせる）/ medium（品質を明確に下げる）/ low（好みや軽微な改善）。
  - location.file には上記の成果物のパスをそのまま書き、行番号は表示された番号を使ってください。
  - severity が high の指摘は location.file・impact・evidence が必須です。medium は location.file と impact が必須です。
  - kind が bug / factual で severity が high / medium の指摘は、verification（読み取りだけで再現・確認できるシェルコマンド）が必須です。
  - 上記を満たさない指摘は無効として捨てられます。
  - 前提の「やらないこと」に当たる指摘は scope を "out" にしてください（後回しリストに入ります）。
  - 以前の指摘をもう一度主張するときは、ref_id にその ID（例: C3）を入れてください。
- 判断に必要な情報が足りない場合は、issues ではなく questions に質問として書いてください。
- verdict: 重大な問題がなければ AGREE、修正が必要なら REVISE。
- AGREE にする場合は、上記「これまでの指摘」の重要度 高・中 のものを一つ残らず resolved_ids に挙げ、それぞれ解決したと言える理由を書いてください。漏れがあると AGREE は無効になります。
- resolved_ids には、REVISE の場合でも解決済みと判断した指摘を挙げてください。`;

function ledgerTable(state) {
  const rows = Object.values(state.issues).filter((i) => i.source === 'round');
  if (!rows.length) return '';
  return `# これまでの指摘\n| ID | 重要度 | 状態 | 場所 | 内容 |\n|---|---|---|---|---|\n${rows
    .map((i) => `| ${i.id} | ${SEVERITY_LABEL[i.severity]} | ${STATUS_LABEL[i.status]}${i.decided_by && i.decided_by !== 'claude' ? `（${i.decided_by === 'judge' ? '第三者判定' : 'ユーザー判断'}）` : ''} | ${fmtLoc(i.location)} | ${normText(i.claim).replace(/\|/g, '/').slice(0, 160)} |`)
    .join('\n')}\n`;
}

function evidenceLine(e) {
  const v = e.verified ? '（スクリプトで照合済み）' : '（未検証）';
  if (e.type === 'quote') return `引用 ${e.file}:${e.line_start}-${e.line_end}${v}: 「${truncate(normText(e.text), 400)}」`;
  if (e.type === 'verify') return `検証 ${e.id}${v}: \`${e.cmd ?? ''}\` → 終了コード ${e.exit ?? '?'}${e.output ? `\n${truncate(e.output, 1500)}` : ''}`;
  if (e.type === 'url') return `出典 ${e.url}${v}: 「${truncate(normText(e.text), 300)}」`;
  return JSON.stringify(e);
}

function previousCycleSection(ctx, round) {
  const { state } = ctx;
  const prev = round - 1;
  const lines = [];
  for (const i of Object.values(state.issues)) {
    for (const h of i.history.filter((x) => x.round === prev && ['decision', 'judge', 'user'].includes(x.type))) {
      if (h.type === 'decision') {
        const label = { accept: '受け入れ', partial: '一部受け入れ', reject: '却下' }[h.decision];
        lines.push(`## ${i.id}（${label}${h.reason_type ? `: ${REASON_LABEL[h.reason_type]}` : ''}）\n${h.rationale}${h.evidence?.length ? `\n根拠:\n${h.evidence.map((e) => `- ${evidenceLine(e)}`).join('\n')}` : ''}`);
      } else if (h.type === 'judge') {
        lines.push(`## ${i.id}（第三者判定: ${{ upheld: '指摘を支持', rejected: '却下を支持', split: '判定が割れたため保留' }[h.result]}）`);
      } else {
        lines.push(`## ${i.id}（ユーザー判断: ${h.decision === 'accept' ? '受け入れ' : '却下'}）${h.note ? `\n${h.note}` : ''}`);
      }
    }
    if (i.status === 'escalated' || i.status === 'held') {
      lines.push(`## ${i.id}（${STATUS_LABEL[i.status]}）`);
    }
  }
  const extra = state.cycle_extra.filter((x) => x.round === prev);
  if (extra.length) lines.push(`## 指摘以外の変更（作者が宣言）\n${extra.map((x) => `- ${x.file}: ${x.reason}`).join('\n')}`);
  const answered = state.questions.filter((q) => q.answer && q.answered_round === prev);
  if (answered.length) lines.push(`## 質問への回答\n${answered.map((q) => `- ${q.id}「${q.text}」→ ${q.answer}（${q.answered_by === 'user' ? 'ユーザー' : '作者'}）`).join('\n')}`);
  return lines.length ? `# 前回の指摘への対応\n${lines.join('\n\n')}\n` : '';
}

function testSection(ctx, testRuns) {
  if (!testRuns?.length) return '';
  return `# テスト・リンターの実行結果（スクリプトが実行）\n${testRuns
    .map((v) => `## ${v.id}: \`${v.cmd}\` → 終了コード ${v.exit}${v.timed_out ? '（時間切れ）' : ''}\n${'```'}\n${truncate(`${v.stdout}\n${v.stderr}`.trim(), 4000)}\n${'```'}`)
    .join('\n\n')}\n`;
}

function buildRoundPrompt(ctx, { round, lens, context, testRuns, first }) {
  const { state } = ctx;
  const s = state.settings;
  const parts = [];
  if (first) {
    parts.push('あなたは独立したレビュアーです。以下の成果物を批評し、誤り・抜け漏れ・より良い代替案を指摘してください。');
    if (lens) parts.push(`今回は「${lens}」の観点に限定して批評してください。`);
    parts.push(`# 前提（依頼者の意図）\n${context}`);
    parts.push(sectionFiles(state, state.artifacts, '成果物（行番号付き）'));
    if (s.refFiles?.length) parts.push(sectionFiles(state, s.refFiles, '参考資料'));
    if (s.diff) {
      const d = gitDiffText(state);
      if (d) parts.push(`# Git の変更差分（HEAD との比較）\n${'```diff'}\n${truncate(d, s.maxInlineBytes / 2)}\n${'```'}`);
    }
  } else {
    parts.push(`第 ${round} ラウンドです。前回の指摘への対応と、更新された成果物を確認してください。${lens ? `引き続き「${lens}」の観点で批評してください。` : ''}`);
    parts.push(previousCycleSection(ctx, round));
    const d = diffVersions(ctx, `r${round - 1}`, `r${round}`);
    parts.push(d ? `# 前回からの変更差分\n${'```diff'}\n${truncate(d, s.maxInlineBytes / 2)}\n${'```'}` : '# 前回からの変更差分\n（変更なし）');
    parts.push(sectionFiles(state, state.artifacts, '現在の成果物（行番号付き）'));
  }
  parts.push(testSection(ctx, testRuns));
  parts.push(ledgerTable(state));
  parts.push(RULES);
  return parts.filter(Boolean).join('\n\n');
}

// ───────────────────────── 指摘の取り込み ─────────────────────────

function validateRawIssue(raw) {
  const errs = [];
  const loc = raw.location || {};
  if (!nonEmpty(raw.claim)) errs.push('claim が空');
  if (raw.severity === 'high' || raw.severity === 'medium') {
    if (!nonEmpty(loc.file)) errs.push('location.file が空');
    if (!nonEmpty(raw.impact)) errs.push('impact が空');
  }
  if (raw.severity === 'high' && !nonEmpty(raw.evidence)) errs.push('evidence が空');
  if (['bug', 'factual'].includes(raw.kind) && raw.severity !== 'low' && !nonEmpty(raw.verification)) errs.push('verification が空');
  return errs;
}

function bigrams(s) {
  const t = String(s).toLowerCase().replace(/\s+/g, '');
  const set = new Set();
  for (let i = 0; i < t.length - 1; i++) set.add(t.slice(i, i + 2));
  return set;
}
function similar(a, b) {
  if ((a.location?.file ?? null) !== (b.location?.file ?? null)) return false;
  const la = a.location || {};
  const lb = b.location || {};
  if (la.line_start != null && lb.line_start != null) {
    const ae = la.line_end ?? la.line_start;
    const be = lb.line_end ?? lb.line_start;
    if (ae + 3 < lb.line_start || be + 3 < la.line_start) return false;
  }
  const A = bigrams(a.claim);
  const B = bigrams(b.claim);
  let inter = 0;
  for (const x of A) if (B.has(x)) inter++;
  const union = A.size + B.size - inter;
  return union > 0 && inter / union >= 0.45;
}

function normalizeRaw(state, raw) {
  const loc = raw.location || {};
  return {
    severity: SEVERITY_RANK[raw.severity] !== undefined ? raw.severity : 'medium',
    kind: raw.kind || 'other',
    scope: raw.scope === 'out' ? 'out' : 'in',
    location: {
      file: normalizeFile(state, loc.file),
      line_start: Number.isInteger(loc.line_start) ? loc.line_start : null,
      line_end: Number.isInteger(loc.line_end) ? loc.line_end : null,
      section: loc.section ?? null,
    },
    claim: raw.claim ?? '',
    impact: raw.impact ?? '',
    evidence: raw.evidence ?? '',
    suggestion: raw.suggestion ?? '',
    verification: nonEmpty(raw.verification) ? raw.verification : null,
  };
}

function escalate(state, issue, reason, round, rejection = null) {
  issue.status = state.settings.judges.length ? 'escalated' : 'held';
  issue.escalation = { reason, round, rejection: rejection ?? issue.escalation?.rejection ?? lastRejection(issue) };
  issue.judge_case = null;
}
function lastRejection(issue) {
  const h = [...issue.history].reverse().find((x) => x.type === 'decision' && x.decision === 'reject');
  return h ? { reason_type: h.reason_type, rationale: h.rationale, evidence: h.evidence } : null;
}
const isBlocking = (state, i) => i.scope === 'in' && state.settings.blockingSeverities.includes(i.severity);

function createIssue(state, item, { round, lens, source = 'round', status }) {
  const id = `C${++state.counters.issue}`;
  const issue = {
    id, ...item, source, lenses: lens ? [lens] : [], raised_rounds: [round], reraise_count: 0, reject_count: 0,
    status: status ?? (item.severity === 'low' || item.scope === 'out' ? 'backlog' : 'open'),
    decided_by: null, accepted_round: null, apply_targets: null, applied: false, verifications: [],
    codex_resolved: {}, history: [{ round, type: 'raised', lens, severity: item.severity }],
  };
  state.issues[id] = issue;
  return issue;
}

function reraise(state, target, item, round, lens) {
  target.history.push({ round, type: 'reraise', lens, severity: item.severity, claim: item.claim });
  target.reraise_count++;
  if (!target.raised_rounds.includes(round)) target.raised_rounds.push(round);
  if (lens && !target.lenses.includes(lens)) target.lenses.push(lens);
  Object.assign(target, {
    severity: item.severity, scope: item.scope, claim: item.claim, impact: item.impact || target.impact,
    evidence: item.evidence || target.evidence, suggestion: item.suggestion || target.suggestion,
    verification: item.verification || target.verification,
  });
  if (item.location?.file) target.location = item.location;
  if (item.severity === 'low' || item.scope === 'out') { target.status = 'backlog'; return; }
  if (target.status === 'rejected') {
    if (target.reject_count >= state.settings.escalation.rejectRepeat) escalate(state, target, 'repeat', round);
    else target.status = 'open';
  } else if (['accepted', 'partial', 'backlog'].includes(target.status)) {
    target.status = 'open';
  }
}

function ingestRound(ctx, round, results) {
  const { state } = ctx;
  const knownBefore = Object.values(state.issues).filter((i) => i.source === 'round' && rank(i.severity) >= 1 && ['accepted', 'partial', 'rejected', 'escalated', 'held'].includes(i.status));
  const summary = { round, at: now(), lenses: {}, new_ids: [], reraised_ids: [], invalid: [], question_ids: [], decisions: null, escalated_ids: [] };
  const touched = new Set();
  const newThisRound = [];
  for (const { lens, data } of results) {
    const lensName = lens || 'main';
    const info = { verdict: data.verdict, summary: data.summary, agree_valid: null };
    const resolved = new Map();
    for (const r of data.resolved_ids || []) {
      if (state.issues[r.id] && nonEmpty(r.reason)) resolved.set(r.id, r.reason);
    }
    for (const [id, reason] of resolved) state.issues[id].codex_resolved[lensName] = { round, reason };
    let blockingInResponse = 0;
    for (const raw of data.issues || []) {
      const item = normalizeRaw(state, raw);
      if (item.scope === 'in' && item.severity !== 'low') {
        const errs = validateRawIssue(raw);
        if (errs.length) { summary.invalid.push({ lens: lensName, claim: truncate(raw.claim || '', 160), errors: errs }); continue; }
      }
      if (isBlocking(state, item)) blockingInResponse++;
      let target = raw.ref_id && state.issues[raw.ref_id] ? state.issues[raw.ref_id] : null;
      if (!target) target = newThisRound.find((i) => similar(i, item));
      if (target && newThisRound.includes(target)) {
        if (lens && !target.lenses.includes(lens)) target.lenses.push(lens);
        if (rank(item.severity) > rank(target.severity)) { target.severity = item.severity; if (target.status === 'backlog' && item.scope === 'in') target.status = 'open'; }
        continue;
      }
      if (!target) target = Object.values(state.issues).find((i) => i.source === 'round' && i.status !== 'backlog' && !newThisRound.includes(i) && similar(i, item));
      if (target) {
        reraise(state, target, item, round, lens);
        touched.add(target.id);
        if (!summary.reraised_ids.includes(target.id)) summary.reraised_ids.push(target.id);
      } else {
        const issue = createIssue(state, item, { round, lens });
        newThisRound.push(issue);
        touched.add(issue.id);
        summary.new_ids.push(issue.id);
      }
    }
    for (const q of data.questions || []) {
      if (!nonEmpty(q)) continue;
      const id = `Q${++state.counters.question}`;
      state.questions.push({ id, round, lens: lensName, text: q, answer: null });
      summary.question_ids.push(id);
    }
    if (data.verdict === 'AGREE') {
      const missing = knownBefore.filter((i) => !resolved.has(i.id)).map((i) => i.id);
      info.agree_valid = missing.length === 0 && blockingInResponse === 0;
      if (!info.agree_valid) {
        info.agree_invalid_reason = missing.length
          ? `解決理由の示されていない指摘があるため合意は無効（W7）: ${missing.join(', ')}`
          : '合意と同時に重要な指摘を出しているため合意は無効（W7）';
      }
    }
    summary.lenses[lensName] = info;
  }

  // 終了判定（T2）
  const touchedIssues = [...touched].map((id) => state.issues[id]);
  const blocking = touchedIssues.filter((i) => isBlocking(state, i) && ['open', 'escalated', 'held'].includes(i.status));
  const allAgree = Object.values(summary.lenses).every((l) => l.verdict === 'AGREE' && l.agree_valid);
  const newBlocking = summary.new_ids.filter((id) => isBlocking(state, state.issues[id]));
  const repeated = round > 1 && blocking.length > 0 && newBlocking.length === 0 && blocking.every((i) => i.reject_count > 0);
  const unchanged = round > 1 && blocking.length > 0 && state.versions[`r${round - 1}`]?.hash === state.versions[`r${round}`]?.hash;
  let terminal = null;
  if (allAgree) terminal = { status: 'converged_agree', reason: 'Codex が有効な合意を示した' };
  else if (blocking.length === 0) terminal = { status: 'converged_no_blocking', reason: '議論を止めるほど重要な未解決の指摘がなくなった' };
  else if (repeated || unchanged) {
    terminal = { status: 'stalemate', reason: repeated ? '却下済みの指摘が新しい論点なしに繰り返された' : '成果物が前の版から変わらないまま指摘が続いた' };
    for (const i of blocking) {
      if (i.status === 'open' && i.reject_count > 0) { escalate(state, i, 'stalemate', round); summary.escalated_ids.push(i.id); }
    }
  } else if (round >= state.settings.rounds) terminal = { status: 'max_rounds', reason: `上限の ${state.settings.rounds} ラウンドに達した` };
  for (const i of touchedIssues) if (i.status === 'escalated' && i.escalation?.round === round && !summary.escalated_ids.includes(i.id)) summary.escalated_ids.push(i.id);
  if (terminal) state.terminal = { ...terminal, round };
  summary.terminal = state.terminal;
  state.rounds.push(summary);
  return summary;
}

// ───────────────────────── 証拠の照合（G1） ─────────────────────────

function checkEvidence(ctx, e) {
  const { dir, state } = ctx;
  if (!e || typeof e !== 'object') return { error: '証拠がオブジェクトではありません' };
  if (e.type === 'quote') {
    if (!nonEmpty(e.file) || !Number.isInteger(e.line_start) || !Number.isInteger(e.line_end) || !nonEmpty(e.text)) {
      return { error: 'quote には file・line_start・line_end・text が必要です' };
    }
    const rel = normalizeFile(state, e.file);
    const src = e.version ? path.join(dir, 'versions', e.version, rel) : path.resolve(state.cwd, rel);
    const text = readIfExists(src);
    const base = { type: 'quote', file: rel, line_start: e.line_start, line_end: e.line_end, text: e.text, version: e.version ?? null };
    if (text == null) return { ...base, verified: false, detail: 'ファイルが見つかりません' };
    const lines = text.split('\n');
    const seg = normText(lines.slice(Math.max(0, e.line_start - 3), e.line_end + 2).join('\n'));
    const q = normText(e.text);
    if (q.length < 8) return { ...base, verified: false, detail: '引用が短すぎます（8文字以上）' };
    return seg.includes(q) ? { ...base, verified: true } : { ...base, verified: false, detail: '指定した行に引用文が見つかりません' };
  }
  if (e.type === 'verify') {
    const v = state.verifies[e.id];
    if (!v) return { type: 'verify', id: e.id, verified: false, detail: 'スクリプトが記録した検証ではありません' };
    const rec = readIfExists(path.join(dir, 'verify', `${e.id}.json`));
    const ok = rec && sha256(JSON.stringify({ ...JSON.parse(rec), hash: undefined })) === JSON.parse(rec).hash;
    return {
      type: 'verify', id: e.id, cmd: v.cmd, exit: v.exit, verified: !!ok,
      output: rec ? truncate(`${JSON.parse(rec).stdout}\n${JSON.parse(rec).stderr}`.trim(), 1500) : '',
      detail: ok ? undefined : '検証記録が改変されています',
    };
  }
  if (e.type === 'url') {
    if (!nonEmpty(e.url) || !nonEmpty(e.text)) return { error: 'url には url と text が必要です' };
    return { type: 'url', url: e.url, text: e.text, verified: false, detail: '外部資料は自動照合できません' };
  }
  return { error: `未知の証拠の種類: ${e.type}` };
}

// ───────────────────────── コマンド ─────────────────────────

async function cmdInit(opts) {
  const cwd = process.cwd();
  const cfg = loadConfig(cwd);
  if (opts.quick && opts.deep) fail('--quick と --deep は同時に指定できません');
  let s = structuredClone(cfg);
  if (opts.quick) s = merge(s, cfg.presets.quick);
  if (opts.deep) s = merge(s, cfg.presets.deep);
  delete s.presets;
  if (opts.rounds) {
    const n = parseInt(opts.rounds, 10);
    if (!(n >= 1 && n <= 10)) fail('--rounds は 1〜10 で指定してください');
    s.rounds = n;
  }
  if (opts.effort) {
    if (!EFFORTS.includes(opts.effort)) fail(`--effort は ${EFFORTS.join(' / ')} のいずれか`);
    s.effort = opts.effort;
  }
  if (opts.model) s.model = opts.model;
  s.interactive = !!opts.interactive;
  s.blind = !!opts.blind;
  s.lenses = opts.lenses ? opts.lenses.split(',').map((x) => x.trim()).filter(Boolean) : [];
  s.code = !!opts.code;
  s.judgeAll = !!opts.judge;
  s.background = !!opts.background;
  s.diff = !!opts.diff || s.code;
  s.mode = opts.quick ? 'quick' : opts.deep ? 'deep' : 'normal';
  s.testCommands = s.code ? [...(cfg.testCommands || []), ...(opts.test || [])] : [...(opts.test || [])];
  const stateStub = { cwd, artifacts: [] };
  s.refFiles = (opts.files || []).map((f) => {
    const rel = toRel(stateStub, f);
    if (!exists(path.resolve(cwd, rel))) fail(`参考ファイルがありません: ${f}`);
    return rel;
  });
  s.images = (opts.image || []).map((f) => {
    const abs = path.resolve(cwd, f);
    if (!exists(abs)) fail(`画像がありません: ${f}`);
    return abs;
  });
  s.addDirs = (opts['add-dir'] || []).map((d) => path.resolve(cwd, d));
  if (s.judges.includes('claude') && !['agent', 'cli'].includes(s.claudeJudgeMode)) fail('claudeJudgeMode は agent か cli');

  const d0 = new Date();
  const pad = (n) => String(n).padStart(2, '0');
  const stamp = `${d0.getFullYear()}${pad(d0.getMonth() + 1)}${pad(d0.getDate())}-${pad(d0.getHours())}${pad(d0.getMinutes())}${pad(d0.getSeconds())}`;
  const id = `${stamp}-${crypto.randomBytes(2).toString('hex')}`;
  const dir = path.resolve(cwd, s.logDir, id);
  fs.mkdirSync(dir, { recursive: true });
  const logRoot = path.resolve(cwd, s.logDir);
  if (!exists(path.join(logRoot, '.gitignore'))) writeText(path.join(logRoot, '.gitignore'), '*\n');

  let artifacts;
  if (opts.artifact?.length) {
    artifacts = opts.artifact.map((f) => toRel(stateStub, f));
  } else if (s.code) {
    const tracked = spawnSync('git', ['-C', cwd, 'diff', '--name-only', 'HEAD'], { encoding: 'utf8' });
    const untracked = spawnSync('git', ['-C', cwd, 'ls-files', '--others', '--exclude-standard'], { encoding: 'utf8' });
    artifacts = [...new Set(`${tracked.stdout || ''}\n${untracked.stdout || ''}`.split('\n').map((x) => x.trim()).filter(Boolean))]
      .filter((f) => !f.startsWith(s.logDir));
    if (!artifacts.length) fail('--code ですが Git の変更が見つかりません。--artifact で対象ファイルを指定してください');
  } else {
    artifacts = [path.relative(cwd, path.join(dir, 'draft.md'))];
  }

  writeText(path.join(dir, 'context.md'), contextTemplate(opts.topic));
  const state = {
    version: 1, id, topic: opts.topic || '', cwd, dir, created_at: now(), updated_at: now(),
    settings: s, artifacts, phase: 'drafting', next_action: '', round: 0, terminal: null, threads: {},
    issues: {}, questions: [], verifies: {}, versions: {}, judge_cases: {}, rounds: [], cycle_extra: [],
    metrics: [], counters: { issue: 0, verify: 0, question: 0, judge: 0 }, final_check: null, blind: null,
    reported_hash: null,
  };
  const ctx = { dir, state };
  saveState(ctx);
  indexAdd({ id, dir, cwd, topic: state.topic, created_at: state.created_at });
  logLine(ctx, `討論を開始: ${state.topic}`);
  out({
    ok: true, id, dir, context_file: path.join(dir, 'context.md'),
    artifacts: artifacts.map((a) => path.resolve(cwd, a)),
    settings: { rounds: s.rounds, mode: s.mode, effort: s.effort, model: s.model ?? '（Codex の設定に従う）', judges: s.judges, claudeJudgeMode: s.claudeJudgeMode, interactive: s.interactive, blind: s.blind, lenses: s.lenses, code: s.code, testCommands: s.testCommands, finalCheck: s.finalCheck },
    next_action: state.next_action,
  });
}

async function cmdBlind(opts) {
  const ctx = loadDebate(opts);
  const { dir, state } = ctx;
  if (state.round > 0) fail('blind は最初のラウンドの前にだけ実行できます');
  const context = checkContext(ctx);
  checkArtifacts(ctx);
  // Claude の初稿を先に固定してから Codex の独自解を見る（初期の推論は Claude 主体）
  if (!state.versions['v0-initial']) snapshot(ctx, 'v0-initial');
  const prompt = [
    'あなたは独立した回答者です。以下の依頼に対して、あなた自身の解答・成果物を作成してください。他者の案は見せません。ファイルは編集できないので、solution に成果物の全文を書いてください。',
    `# 前提（依頼者の意図）\n${context}`,
    state.settings.refFiles?.length ? sectionFiles(state, state.settings.refFiles, '参考資料') : '',
  ].filter(Boolean).join('\n\n');
  guardSecrets(ctx, opts, prompt, 'blind');
  const res = await callCodex(ctx, { label: 'blind', prompt, schema: 'blind.json', ephemeral: true, images: state.settings.images, round: 0 });
  const md = `# Codex の独自解（Claude の案を見せずに作成）\n\n${res.data.solution}\n\n## 要点\n${res.data.key_points.map((x) => `- ${x}`).join('\n')}\n\n## 前提にしたこと\n${res.data.assumptions.map((x) => `- ${x}`).join('\n')}\n`;
  const f = path.join(dir, 'blind', 'codex_solution.md');
  writeText(f, md);
  state.blind = { at: now(), file: f, v0_hash: state.versions['v0-initial'].hash };
  logLine(ctx, 'Codex の独自解を取得（M3）');
  saveState(ctx);
  out({ ok: true, codex_solution: f, note: 'Claude の初稿は v0-initial として固定済み。独自解と比べ、取り入れる点があればドラフトに反映してから round を実行する', next_action: state.next_action });
}

async function runTests(ctx) {
  const { state } = ctx;
  const runs = [];
  for (const cmd of state.settings.testCommands || []) runs.push(await recordVerify(ctx, { cmd, issue: null, auto: true }));
  return runs;
}

async function recordVerify(ctx, { cmd, issue, auto = false }) {
  const { dir, state } = ctx;
  const s = state.settings;
  const id = `V${++state.counters.verify}`;
  const r = await runProc('/bin/sh', ['-c', cmd], { cwd: state.cwd, timeoutMs: s.verifyTimeoutSec * 1000 });
  const rec = {
    id, cmd, issue: issue ?? null, round: state.round, auto, at: now(), exit: r.code, timed_out: r.timedOut,
    duration_ms: r.duration, stdout: truncate(r.stdout, s.verifyOutputBytes), stderr: truncate(r.stderr, s.verifyOutputBytes),
  };
  rec.hash = sha256(JSON.stringify({ ...rec, hash: undefined }));
  writeJSON(path.join(dir, 'verify', `${id}.json`), rec);
  state.verifies[id] = { id, cmd, issue: rec.issue, round: rec.round, auto, exit: rec.exit, timed_out: rec.timed_out, at: rec.at };
  if (issue && state.issues[issue]) state.issues[issue].verifications.push(id);
  return rec;
}

async function cmdRound(opts) {
  const ctx = loadDebate(opts);
  const { dir, state } = ctx;
  const s = state.settings;
  refreshPhase(state);
  if (state.terminal) fail(`議論は既に終了しています（${state.terminal.status}）。次は: ${state.next_action}`);
  if (state.phase === 'awaiting_response') fail('未回答の指摘があります。先に respond を実行してください', { open: Object.values(state.issues).filter((i) => i.status === 'open').map((i) => i.id) });
  if (state.phase === 'awaiting_judge') fail('判定待ちの指摘があります。先に judge run / judge resolve を実行してください');
  if (state.phase === 'awaiting_user') fail('保留中の指摘があります。ユーザーに確認して decide で記録してください');
  const context = checkContext(ctx);
  checkArtifacts(ctx);
  if (state.round > 0) {
    const chk = applyCheck(ctx);
    if (chk.errors.length) fail('前回の対応の反映確認で問題が見つかりました', { errors: chk.errors });
  }
  const round = state.round + 1;
  if (round > s.rounds) fail('上限のラウンド数に達しています');
  snapshot(ctx, `r${round}`);
  if (!state.versions['v0-initial']) snapshot(ctx, 'v0-initial');
  const testRuns = s.code ? await runTests(ctx) : [];
  const lenses = s.lenses.length ? s.lenses : [null];
  const jobs = lenses.map((lens) => {
    const key = lens || 'main';
    const threadId = state.threads[key] || null;
    const prompt = buildRoundPrompt(ctx, { round, lens, context, testRuns, first: !threadId });
    return { lens, key, threadId, prompt };
  });
  for (const j of jobs) guardSecrets(ctx, opts, j.prompt, `round${round}-${j.key}`);
  logLine(ctx, `ラウンド ${round} を開始（${jobs.map((j) => j.key).join(', ')}）`);
  saveState(ctx);

  // 一部の観点だけ成功した場合に二重送信しないよう、成功結果を保存して再利用する
  const results = await Promise.all(jobs.map(async (j) => {
    const cacheFile = path.join(dir, 'rounds', `r${round}`, `${j.key}.result.json`);
    const promptHash = sha256(j.prompt);
    if (exists(cacheFile)) {
      const c = readJSON(cacheFile);
      if (c.prompt_hash === promptHash) return { ...c, cached: true };
    }
    const res = await callCodex(ctx, {
      label: `round${round}-${j.key}`, prompt: j.prompt, schema: 'critique.json', threadId: j.threadId,
      images: j.threadId ? [] : s.images, round,
    });
    const rec = { lens: j.lens, key: j.key, data: res.data, threadId: res.threadId, prompt_hash: promptHash };
    writeJSON(cacheFile, rec);
    return rec;
  }).map((p) => p.catch((e) => ({ error: e }))));
  const failed = results.filter((r) => r.error);
  if (failed.length) {
    saveState(ctx);
    const e = failed[0].error;
    throw e instanceof UserError ? e : new UserError(String(e));
  }
  for (const r of results) if (r.threadId) state.threads[r.key] = r.threadId;
  state.round = round;
  const summary = ingestRound(ctx, round, results.map((r) => ({ lens: r.lens, data: r.data })));
  delete state.failed;
  logLine(ctx, `ラウンド ${round} 完了: 新規 ${summary.new_ids.length} / 再提起 ${summary.reraised_ids.length} / 無効 ${summary.invalid.length}${state.terminal ? ` / 終了: ${state.terminal.status}` : ''}`);
  saveState(ctx);

  const open = Object.values(state.issues).filter((i) => i.status === 'open');
  const template = {
    decisions: open.map((i) => ({ id: i.id, decision: '', reason_type: null, rationale: '', evidence: [], applied_at: [] })),
    extra_changes: [],
  };
  const templateFile = path.join(dir, 'rounds', `r${round}`, 'response.template.json');
  writeJSON(templateFile, template);
  out({
    ok: true, round, terminal: state.terminal, phase: state.phase,
    lenses: summary.lenses,
    issues_to_decide: open.map(issueView),
    escalated: summary.escalated_ids,
    backlog_added: Object.values(state.issues).filter((i) => i.status === 'backlog' && i.raised_rounds.includes(round)).map((i) => ({ id: i.id, severity: i.severity, claim: truncate(i.claim, 200) })),
    invalid_issues: summary.invalid,
    questions: state.questions.filter((q) => q.round === round),
    tests: testRuns.map((v) => ({ id: v.id, cmd: v.cmd, exit: v.exit })),
    response_file: path.join(dir, 'rounds', `r${round}`, 'response.json'),
    response_template: templateFile,
    next_action: state.next_action,
  });
}

function issueView(i) {
  return {
    id: i.id, severity: i.severity, kind: i.kind, location: fmtLoc(i.location), claim: i.claim, impact: i.impact,
    evidence: i.evidence, suggestion: i.suggestion, verification: i.verification, reraise_count: i.reraise_count,
    reject_count: i.reject_count, lenses: i.lenses, status: i.status,
  };
}

async function cmdVerify(opts) {
  const ctx = loadDebate(opts);
  const { state } = ctx;
  const issue = opts.issue ? state.issues[opts.issue] : null;
  if (opts.issue && !issue) fail(`指摘 ${opts.issue} がありません`);
  const cmd = opts.cmd || issue?.verification;
  if (!nonEmpty(cmd)) fail('--cmd を指定してください（指摘に verification がありません）');
  const rec = await recordVerify(ctx, { cmd, issue: issue?.id });
  logLine(ctx, `検証 ${rec.id}: ${cmd} → ${rec.exit}`);
  saveState(ctx);
  out({ ok: true, ...rec, evidence_ref: { type: 'verify', id: rec.id } });
}

function isSelfQuote(issue, e) {
  const loc = issue.location;
  if (!loc?.file || e.file !== loc.file) return false;
  if (loc.line_start == null) return true;
  return e.line_start <= (loc.line_end ?? loc.line_start) && e.line_end >= loc.line_start;
}

function applyDecision(state, issue, d, by, round) {
  if (d.decision === 'reject') {
    issue.status = 'rejected';
    issue.reject_count++;
  } else {
    issue.status = d.decision === 'accept' ? 'accepted' : 'partial';
    issue.accepted_round = round;
    issue.apply_targets = (d.applied_at || []).length
      ? d.applied_at.map((t) => ({ file: normalizeFile(state, t.file), line_start: t.line_start ?? null, line_end: t.line_end ?? t.line_start ?? null }))
      : [issue.location];
    issue.applied = false;
  }
  issue.decided_by = by;
  issue.history.push({
    round, type: 'decision', by, decision: d.decision, reason_type: d.reason_type ?? null, duplicate_of: d.duplicate_of ?? null,
    rationale: d.rationale, evidence: d._evidence ?? [], at: now(),
  });
}

async function cmdRespond(opts) {
  const ctx = loadDebate(opts);
  const { dir, state } = ctx;
  const round = state.round;
  if (round === 0) fail('まだラウンドが実行されていません');
  const file = opts.file ? path.resolve(opts.file) : path.join(dir, 'rounds', `r${round}`, 'response.json');
  if (!exists(file)) fail(`回答ファイルがありません: ${file}`, { template: path.join(dir, 'rounds', `r${round}`, 'response.template.json') });
  let resp;
  try { resp = readJSON(file); } catch (e) { fail(`回答ファイルが JSON として読めません: ${e.message}`); }
  const errors = [];
  const decisions = Array.isArray(resp.decisions) ? resp.decisions : [];
  if (!Array.isArray(resp.decisions)) errors.push('decisions が配列ではありません');
  const seen = new Set();
  for (const d of decisions) {
    const issue = state.issues[d.id];
    if (!issue) { errors.push(`${d.id}: 存在しない指摘 ID`); continue; }
    if (seen.has(d.id)) errors.push(`${d.id}: 回答が重複しています`);
    seen.add(d.id);
    if (!['open', 'backlog', 'final_open'].includes(issue.status)) errors.push(`${d.id}: 状態「${STATUS_LABEL[issue.status]}」の指摘には回答できません`);
    if (!['accept', 'partial', 'reject'].includes(d.decision)) { errors.push(`${d.id}: decision は accept / partial / reject のいずれか`); continue; }
    if (!nonEmpty(d.rationale)) errors.push(`${d.id}: rationale が空です`);
    d._evidence = (Array.isArray(d.evidence) ? d.evidence : []).map((e) => checkEvidence(ctx, e));
    for (const e of d._evidence) if (e.error) errors.push(`${d.id}: 証拠の形式エラー: ${e.error}`);
    // 却下の証拠として、指摘された箇所そのものの引用は反証にならない（その記述が正しいかが争点のため）
    if (d.decision === 'reject') {
      for (const e of d._evidence) {
        if (e.type === 'quote' && e.verified && isSelfQuote(issue, e)) {
          e.verified = false;
          e.detail = '指摘された箇所そのものの引用は、却下の証拠になりません';
        }
      }
    }
    if (d.decision === 'reject') {
      if (!REASON_TYPES.includes(d.reason_type)) errors.push(`${d.id}: 却下には reason_type（${REASON_TYPES.join(' / ')}）が必要です（W1）`);
      if (d.reason_type === 'duplicate' && !state.issues[d.duplicate_of]) errors.push(`${d.id}: duplicate_of に重複先の指摘 ID を入れてください`);
      if (!(d._evidence.length || d.reason_type === 'duplicate')) errors.push(`${d.id}: 却下には証拠（quote / verify / url）が必要です（W1）`);
    } else if (issue.severity === 'high' && !d._evidence.some((e) => e.verified)) {
      errors.push(`${d.id}: 重要度 高 の指摘を受け入れるには、指摘が正しいことを示す照合済みの証拠（quote か verify）が必要です（W4）`);
    }
  }
  const missing = Object.values(state.issues).filter((i) => i.status === 'open' && !seen.has(i.id)).map((i) => i.id);
  if (missing.length) errors.push(`回答のない指摘があります（T3）: ${missing.join(', ')}`);
  const extra = Array.isArray(resp.extra_changes) ? resp.extra_changes : [];
  for (const x of extra) if (!nonEmpty(x?.file) || !nonEmpty(x?.reason)) errors.push('extra_changes の各項目には file と reason が必要です');
  if (errors.length) fail('回答ファイルに問題があります。修正して再実行してください', { errors });

  // 仮に適用して反映確認（W4・G4）。問題があれば何も記録しない
  const trial = structuredClone(state);
  for (const d of decisions) applyDecision(trial, trial.issues[d.id], d, 'claude', round);
  for (const x of extra) trial.cycle_extra.push({ round, file: normalizeFile(trial, x.file), reason: x.reason });
  const chk = applyCheck({ dir, state: trial });
  if (chk.errors.length) fail('修正の反映確認で問題が見つかりました。何も記録していません', { errors: chk.errors });

  for (const d of decisions) applyDecision(state, state.issues[d.id], d, 'claude', round);
  for (const x of extra) state.cycle_extra.push({ round, file: normalizeFile(state, x.file), reason: x.reason });
  for (const i of Object.values(trial.issues)) if (i.applied) state.issues[i.id].applied = true;

  // 却下の扱い（W1・W2・W3・M6）
  const escalated = [];
  const rejects = decisions.filter((d) => d.decision === 'reject');
  for (const d of rejects) {
    const issue = state.issues[d.id];
    const verified = d._evidence.some((e) => e.verified) || (d.reason_type === 'duplicate' && !!state.issues[d.duplicate_of]);
    const rejection = { reason_type: d.reason_type, rationale: d.rationale, evidence: d._evidence };
    let reason = null;
    if (!verified) reason = 'no_evidence';
    else if (issue.severity === 'high' && issue.scope === 'in') reason = 'high_reject';
    else if (state.terminal && state.settings.judgeAll && rank(issue.severity) >= 1) reason = 'judge_all';
    if (reason) { escalate(state, issue, reason, round, rejection); escalated.push({ id: issue.id, reason }); }
  }
  const esc = state.settings.escalation;
  const rate = decisions.length ? rejects.length / decisions.length : 0;
  if (decisions.length >= esc.minDecisionsForRate && rate > esc.rejectRateThreshold) {
    for (const d of rejects) {
      const issue = state.issues[d.id];
      if (issue.status === 'rejected' && rank(issue.severity) >= 1) {
        escalate(state, issue, 'reject_rate', round, { reason_type: d.reason_type, rationale: d.rationale, evidence: d._evidence });
        escalated.push({ id: issue.id, reason: 'reject_rate' });
      }
    }
  }
  const sum = state.rounds.find((r) => r.round === round);
  if (sum) {
    sum.decisions = {
      accept: decisions.filter((d) => d.decision === 'accept').length,
      partial: decisions.filter((d) => d.decision === 'partial').length,
      reject: rejects.length,
    };
    sum.escalated_ids = [...new Set([...(sum.escalated_ids || []), ...escalated.map((e) => e.id)])];
  }
  writeJSON(path.join(dir, 'rounds', `r${round}`, 'response.accepted.json'), resp);
  logLine(ctx, `回答を記録（受け入れ ${sum?.decisions?.accept ?? 0} / 一部 ${sum?.decisions?.partial ?? 0} / 却下 ${rejects.length}、判定役へ ${escalated.length}）`);
  saveState(ctx);
  out({
    ok: true, round,
    recorded: decisions.map((d) => ({ id: d.id, decision: d.decision, status: state.issues[d.id].status })),
    escalated, held: Object.values(state.issues).filter((i) => i.status === 'held').map((i) => i.id),
    unverified_evidence: decisions.flatMap((d) => d._evidence.filter((e) => !e.verified).map((e) => ({ id: d.id, type: e.type, detail: e.detail }))),
    phase: state.phase, next_action: state.next_action,
  });
}

function buildJudgePacket(ctx, issue, nonce, swap) {
  const { state, dir } = ctx;
  const context = readIfExists(path.join(dir, 'context.md')) ?? '';
  const pro = [
    `この箇所には問題がある（主張された重要度: ${SEVERITY_LABEL[issue.severity]}）。`,
    `主張: ${issue.claim}`,
    issue.impact && `影響: ${issue.impact}`,
    issue.evidence && `根拠: ${issue.evidence}`,
    issue.suggestion && `修正案: ${issue.suggestion}`,
  ].filter(Boolean).join('\n');
  const rej = issue.escalation?.rejection;
  const con = rej
    ? [
      `この指摘は当たらない（理由: ${REASON_LABEL[rej.reason_type] ?? '不明'}）。`,
      `説明: ${rej.rationale}`,
      rej.evidence?.length ? `根拠:\n${rej.evidence.map((e) => `- ${evidenceLine(e)}`).join('\n')}` : '根拠: なし',
    ].join('\n')
    : 'この指摘は当たらない（反論の詳細は記録されていない）。';
  const X = swap ? con : pro;
  const Y = swap ? pro : con;
  let excerpt = '';
  const f = issue.location?.file;
  const text = f ? readIfExists(path.resolve(state.cwd, f)) : null;
  if (text != null) {
    const lines = text.split('\n');
    const s0 = Math.max(1, (issue.location.line_start ?? 1) - 15);
    const e0 = Math.min(lines.length, (issue.location.line_end ?? issue.location.line_start ?? 40) + 15);
    excerpt = `# 該当箇所（${f} の ${s0}〜${e0} 行目、現在の版）\n${lines.slice(s0 - 1, e0).map((l, k) => `${String(s0 + k).padStart(5)}| ${l}`).join('\n')}`;
  }
  const verifs = issue.verifications.map((id) => state.verifies[id]).filter(Boolean);
  const verifText = verifs.length
    ? `# この指摘に関してスクリプトが実行した検証\n${verifs.map((v) => {
      const rec = readIfExists(path.join(dir, 'verify', `${v.id}.json`));
      const r = rec ? JSON.parse(rec) : {};
      return `- ${v.id}: \`${v.cmd}\` → 終了コード ${v.exit}\n${truncate(`${r.stdout ?? ''}\n${r.stderr ?? ''}`.trim(), 1500)}`;
    }).join('\n')}`
    : '';
  return [
    'あなたは中立の判定者です。ある成果物の一箇所について、2つの主張が対立しています。どちらの主張を誰が書いたかは伏せてあり、提示の順番も無作為です。',
    '与えられた情報だけで判断してください。証拠に「照合済み」とあるものはスクリプトが実物と照らし合わせて確認済み、「未検証」とあるものは確認されていません。',
    `# 前提（依頼者の意図）\n${context}`,
    excerpt,
    `# 主張X\n${X}`,
    `# 主張Y\n${Y}`,
    verifText,
    `# 判定\n- winner: X と Y のどちらがより正しいか。情報が足りず判断できなければ undecidable。\n- severity: この問題の重要度を high / medium / low で評価し直してください（問題がない場合は low）。\n- confidence: 0〜1。\n- reasoning: 判断の理由。\n- case_nonce には次の値をそのまま入れてください: ${nonce}`,
  ].filter(Boolean).join('\n\n');
}

function validateJudgeVerdict(c, data) {
  const errs = [];
  if (!data || typeof data !== 'object') return ['判定が JSON オブジェクトではありません'];
  if (data.case_nonce !== c.nonce) errs.push('case_nonce が一致しません');
  if (!['X', 'Y', 'undecidable'].includes(data.winner)) errs.push('winner は X / Y / undecidable');
  if (SEVERITY_RANK[data.severity] === undefined) errs.push('severity は high / medium / low');
  if (!nonEmpty(data.reasoning)) errs.push('reasoning が空');
  return errs;
}

async function cmdJudge(opts) {
  const sub = opts._[1];
  const ctx = loadDebate(opts);
  const { dir, state } = ctx;
  const s = state.settings;
  if (sub === 'run') {
    if (!s.judges.length) fail('判定役が設定されていません（--quick など）。保留の指摘は decide で記録してください');
    for (const issue of Object.values(state.issues).filter((i) => i.status === 'escalated' && !i.judge_case)) {
      const cid = `J${++state.counters.judge}`;
      const nonce = crypto.randomBytes(6).toString('hex');
      const swap = crypto.randomInt(2) === 1;
      const packet = buildJudgePacket(ctx, issue, nonce, swap);
      const packetFile = path.join(dir, 'judge', cid, 'packet.md');
      writeText(packetFile, packet);
      state.judge_cases[cid] = {
        id: cid, issue: issue.id, nonce, mapping: swap ? { X: 'rejection', Y: 'issue' } : { X: 'issue', Y: 'rejection' },
        packet: packetFile, verdicts: {}, reason: issue.escalation?.reason, created_round: state.round, resolved: null,
      };
      issue.judge_case = cid;
    }
    const pending = Object.values(state.judge_cases).filter((c) => !c.resolved);
    for (const c of pending) guardSecrets(ctx, opts, readIfExists(c.packet), `judge-${c.id}`);
    saveState(ctx);
    if (s.judges.includes('codex')) {
      const res = await Promise.all(pending.filter((c) => !c.verdicts.codex).map(async (c) => {
        try {
          const r = await callCodex(ctx, { label: `judge-${c.id}-codex`, prompt: readIfExists(c.packet), schema: 'judge.json', ephemeral: true, round: state.round });
          const errs = validateJudgeVerdict(c, r.data);
          if (errs.length) return { id: c.id, error: errs.join(', ') };
          c.verdicts.codex = r.data;
          return { id: c.id, ok: true };
        } catch (e) { return { id: c.id, error: e.message }; }
      }));
      const bad = res.filter((r) => r.error);
      saveState(ctx);
      if (bad.length) fail('Codex 判定役の呼び出しに失敗した判定があります。judge run を再実行してください', { failures: bad });
    }
    const agentTasks = [];
    if (s.judges.includes('claude')) {
      for (const c of pending.filter((x) => !x.verdicts.claude)) {
        if (s.claudeJudgeMode === 'cli') {
          const data = await callClaudeCli(ctx, { label: `judge-${c.id}-claude`, prompt: readIfExists(c.packet), schema: 'judge.json', round: state.round });
          const errs = validateJudgeVerdict(c, data);
          if (errs.length) fail(`Claude 判定役の出力が不正です（${c.id}）: ${errs.join(', ')}`);
          c.verdicts.claude = data;
        } else {
          const outFile = path.join(dir, 'judge', c.id, 'claude.json');
          agentTasks.push({
            case: c.id, packet: c.packet, output_file: outFile,
            submit_command: `node "${SCRIPT_PATH}" judge submit --id ${state.id} --case ${c.id} --file "${outFile}"`,
          });
        }
      }
    }
    saveState(ctx);
    out({
      ok: true,
      cases: pending.map((c) => ({ case: c.id, issue: c.issue, reason: c.reason, have: Object.keys(c.verdicts) })),
      claude_agent_tasks: agentTasks,
      next_action: agentTasks.length ? '各 claude_agent_tasks について、議論を知らない新しいサブエージェントに packet を判定させ、submit_command を実行させる。全件そろったら judge resolve' : 'judge resolve を実行する',
    });
  } else if (sub === 'submit') {
    const c = state.judge_cases[opts.case];
    if (!c) fail(`判定 ${opts.case} がありません`);
    if (c.resolved) fail('この判定は既に確定しています');
    if (!opts.file || !exists(opts.file)) fail('--file に判定 JSON を指定してください');
    let data;
    try { data = readJSON(opts.file); } catch (e) { fail(`判定 JSON が読めません: ${e.message}`); }
    const errs = validateJudgeVerdict(c, data);
    if (errs.length) fail('判定 JSON が不正です', { errors: errs });
    c.verdicts.claude = { ...data, submitted_at: now() };
    saveState(ctx);
    out({ ok: true, case: c.id });
  } else if (sub === 'resolve') {
    const results = [];
    const waiting = [];
    for (const c of Object.values(state.judge_cases).filter((x) => !x.resolved)) {
      const missingJudges = s.judges.filter((j) => !c.verdicts[j]);
      if (missingJudges.length) { waiting.push({ case: c.id, missing: missingJudges }); continue; }
      const issue = state.issues[c.issue];
      const outcomes = s.judges.map((j) => (c.verdicts[j].winner === 'undecidable' ? 'undecidable' : c.mapping[c.verdicts[j].winner]));
      let result = 'split';
      if (outcomes.every((o) => o === 'issue')) result = 'upheld';
      else if (outcomes.every((o) => o === 'rejection')) result = 'rejected';
      // G5: 判定役がそろって重要度を下げた場合だけ下げる（上げはしない）
      const judgedSev = maxSeverity(s.judges.map((j) => c.verdicts[j].severity));
      const before = issue.severity;
      if (rank(judgedSev) < rank(issue.severity)) issue.severity = judgedSev;
      if (issue.severity === 'low') {
        issue.status = 'backlog';
      } else if (result === 'upheld') {
        issue.status = 'accepted';
        issue.decided_by = 'judge';
        issue.accepted_round = state.round;
        issue.apply_targets = [issue.location];
        issue.applied = false;
      } else if (result === 'rejected') {
        issue.status = 'rejected';
        issue.decided_by = 'judge';
        issue.judge_confirmed = true;
      } else {
        issue.status = 'held';
      }
      issue.history.push({ round: state.round, type: 'judge', case: c.id, result, outcomes, severity_before: before, severity_after: issue.severity, at: now() });
      c.resolved = { result, outcomes, severity: issue.severity, at: now() };
      results.push({ case: c.id, issue: issue.id, result, severity: issue.severity, status: issue.status });
    }
    logLine(ctx, `判定を確定: ${results.map((r) => `${r.issue}=${r.result}`).join(', ') || 'なし'}`);
    saveState(ctx);
    out({
      ok: waiting.length === 0, results, waiting,
      must_apply: results.filter((r) => r.status === 'accepted').map((r) => r.issue),
      held: Object.values(state.issues).filter((i) => i.status === 'held').map((i) => i.id),
      phase: state.phase, next_action: state.next_action,
    });
  } else {
    fail('judge のサブコマンドは run / submit / resolve');
  }
}

async function cmdDecide(opts) {
  const ctx = loadDebate(opts);
  const { state } = ctx;
  const issue = state.issues[opts.issue];
  if (!issue) fail(`指摘 ${opts.issue} がありません`);
  if (!['accept', 'reject'].includes(opts.decision)) fail('--decision は accept か reject');
  if (!['held', 'escalated', 'open', 'backlog', 'final_open', 'rejected'].includes(issue.status)) fail(`状態「${STATUS_LABEL[issue.status]}」の指摘は decide できません`);
  if (opts.decision === 'accept') {
    issue.status = 'accepted';
    issue.accepted_round = state.round;
    issue.apply_targets = [issue.location];
    issue.applied = false;
  } else {
    issue.status = 'rejected';
  }
  issue.decided_by = 'user';
  issue.history.push({ round: state.round, type: 'user', decision: opts.decision, note: opts.note ?? null, at: now() });
  if (issue.judge_case && state.judge_cases[issue.judge_case] && !state.judge_cases[issue.judge_case].resolved) {
    state.judge_cases[issue.judge_case].resolved = { result: 'user', at: now() };
  }
  logLine(ctx, `ユーザー判断: ${issue.id} = ${opts.decision}`);
  saveState(ctx);
  out({ ok: true, issue: issue.id, status: issue.status, phase: state.phase, next_action: state.next_action });
}

async function cmdAnswer(opts) {
  const ctx = loadDebate(opts);
  const q = ctx.state.questions.find((x) => x.id === opts.q);
  if (!q) fail(`質問 ${opts.q} がありません`);
  if (!nonEmpty(opts.text)) fail('--text が必要です');
  Object.assign(q, { answer: opts.text, answered_by: opts.by === 'claude' ? 'claude' : 'user', answered_round: ctx.state.round, answered_at: now() });
  saveState(ctx);
  out({ ok: true, question: q });
}

async function cmdDeclareChange(opts) {
  const ctx = loadDebate(opts);
  if (!nonEmpty(opts.file) || !nonEmpty(opts.reason)) fail('--file と --reason が必要です');
  ctx.state.cycle_extra.push({ round: ctx.state.round, file: normalizeFile(ctx.state, opts.file), reason: opts.reason });
  saveState(ctx);
  out({ ok: true });
}

async function cmdAddArtifact(opts) {
  const ctx = loadDebate(opts);
  const { dir, state } = ctx;
  const rel = toRel(state, opts.path || '');
  if (state.artifacts.includes(rel)) fail('既に成果物に含まれています');
  state.artifacts.push(rel);
  const src = path.resolve(state.cwd, rel);
  // 編集前の内容を、現在のラウンドと初稿の版に加える（反映確認の基準にする）
  for (const label of [`r${state.round}`, 'v0-initial']) {
    const vdir = path.join(dir, 'versions', label);
    if (!exists(vdir) || !exists(src)) continue;
    const dst = path.join(vdir, rel);
    fs.mkdirSync(path.dirname(dst), { recursive: true });
    fs.copyFileSync(src, dst);
  }
  saveState(ctx);
  out({ ok: true, artifacts: state.artifacts });
}

async function cmdFinalCheck(opts) {
  const ctx = loadDebate(opts);
  const { state } = ctx;
  const s = state.settings;
  refreshPhase(state);
  if (!state.terminal) fail('議論がまだ終わっていません');
  if (['awaiting_response', 'awaiting_judge', 'awaiting_user'].includes(state.phase)) fail(`先に済ませることがあります: ${state.next_action}`);
  const chk = applyCheck(ctx);
  if (chk.errors.length) fail('反映確認で問題が見つかりました', { errors: chk.errors });
  if (!s.finalCheck) {
    state.final_check = { skipped: true, at: now() };
    saveState(ctx);
    out({ ok: true, skipped: true, next_action: state.next_action });
    return;
  }
  const context = checkContext(ctx);
  snapshot(ctx, 'final');
  const prompt = [
    'あなたは独立したレビュアーです。これは最終確認です。以下の成果物を初めて見る立場で批評し、残っている誤り・抜け漏れを指摘してください。',
    `# 前提（依頼者の意図）\n${context}`,
    sectionFiles(state, state.artifacts, '成果物（行番号付き）'),
    s.refFiles?.length ? sectionFiles(state, s.refFiles, '参考資料') : '',
    RULES,
  ].filter(Boolean).join('\n\n');
  guardSecrets(ctx, opts, prompt, 'final-check');
  const res = await callCodex(ctx, { label: 'final-check', prompt, schema: 'critique.json', ephemeral: true, images: s.images, round: state.round });
  const newIds = [];
  const invalid = [];
  for (const raw of res.data.issues || []) {
    const item = normalizeRaw(state, raw);
    if (item.scope === 'in' && item.severity !== 'low') {
      const errs = validateRawIssue(raw);
      if (errs.length) { invalid.push({ claim: truncate(raw.claim || '', 160), errors: errs }); continue; }
    }
    const status = item.severity === 'low' || item.scope === 'out' ? 'backlog' : 'final_open';
    const issue = createIssue(state, item, { round: state.round, lens: 'final', source: 'final_check', status });
    newIds.push(issue.id);
  }
  state.final_check = { at: now(), verdict: res.data.verdict, summary: res.data.summary, new_ids: newIds, invalid, hash: state.versions.final.hash };
  logLine(ctx, `最終確認: ${res.data.verdict}、新規 ${newIds.length}`);
  saveState(ctx);
  out({
    ok: true, verdict: res.data.verdict, summary: res.data.summary,
    new_issues: newIds.map((id) => issueView(state.issues[id])), invalid,
    phase: state.phase, next_action: state.next_action,
  });
}

function sumUsage(metrics, agent) {
  return metrics.filter((m) => m.agent === agent).reduce((a, m) => ({
    calls: a.calls + 1,
    duration_ms: a.duration_ms + (m.duration_ms || 0),
    input: a.input + (m.usage?.input || 0),
    output: a.output + (m.usage?.output || 0),
    cost_usd: a.cost_usd + (m.cost_usd || 0),
  }), { calls: 0, duration_ms: 0, input: 0, output: 0, cost_usd: 0 });
}

async function cmdReport(opts) {
  const ctx = loadDebate(opts);
  const { dir, state } = ctx;
  refreshPhase(state);
  if (!state.terminal) fail('議論がまだ終わっていません');
  if (['awaiting_response', 'awaiting_judge', 'awaiting_user', 'finalizing'].includes(state.phase)) fail(`先に済ませることがあります: ${state.next_action}`);
  const chk = applyCheck(ctx);
  snapshot(ctx, 'final');
  const finalChanged = state.final_check?.hash && state.final_check.hash !== state.versions.final.hash;
  const diff = diffVersions(ctx, 'v0-initial', 'final');
  writeText(path.join(dir, 'diff.patch'), diff);
  const issues = Object.values(state.issues);
  const verifiedIssue = (i) => i.verifications.length > 0 || i.history.some((h) => h.evidence?.some((e) => e.type === 'verify' && e.verified));
  const integrity = Object.keys(state.verifies).filter((id) => {
    const rec = readIfExists(path.join(dir, 'verify', `${id}.json`));
    if (!rec) return true;
    const r = JSON.parse(rec);
    return sha256(JSON.stringify({ ...r, hash: undefined })) !== r.hash;
  });
  const unresolved = issues.filter((i) => ['held', 'final_open', 'open', 'escalated'].includes(i.status));
  const codexUse = sumUsage(state.metrics, 'codex');
  const claudeUse = sumUsage(state.metrics, 'claude');
  const T = {
    converged_agree: '合意（Codex が全ての重要な指摘の解決を確認）',
    converged_no_blocking: '収束（重要な未解決の指摘なし）',
    stalemate: '平行線（判定役に回して決着）',
    max_rounds: '上限ラウンドに到達',
  };
  const md = [];
  md.push(`# 議論レポート: ${state.topic || state.id}`);
  md.push(`- ID: ${state.id}\n- 開始: ${state.created_at}\n- 結論: **${T[state.terminal.status] ?? state.terminal.status}**（${state.terminal.reason}）\n- ラウンド数: ${state.round} / ${state.settings.rounds}\n- モード: ${state.settings.mode}${state.settings.lenses.length ? `、観点: ${state.settings.lenses.join(', ')}` : ''}${state.settings.code ? '、コード' : ''}${state.settings.blind ? '、独自解比較' : ''}`);
  md.push('## 各ラウンドの要約');
  for (const r of state.rounds) {
    const lens = Object.entries(r.lenses).map(([k, v]) => `${k === 'main' ? '' : `${k}: `}${v.verdict}${v.agree_valid === false ? '（合意無効）' : ''}`).join(' / ');
    const sev = (ids) => ids.map((id) => `${id}(${SEVERITY_LABEL[state.issues[id]?.severity] ?? '?'})`).join(' ');
    md.push(`### ラウンド ${r.round}\n- Codex の判定: ${lens}\n- 新しい指摘: ${sev(r.new_ids) || 'なし'}\n- 再提起: ${sev(r.reraised_ids) || 'なし'}\n- 形式不備で無効: ${r.invalid.length} 件\n- Claude の対応: ${r.decisions ? `受け入れ ${r.decisions.accept} / 一部 ${r.decisions.partial} / 却下 ${r.decisions.reject}` : '—'}\n- 判定役へ: ${r.escalated_ids?.join(', ') || 'なし'}${Object.values(r.lenses).map((v) => (v.agree_invalid_reason ? `\n- ${v.agree_invalid_reason}` : '')).join('')}`);
  }
  md.push(`## 指摘一覧\n| ID | 重要度 | 種類 | 場所 | 内容 | 状態 | 決定 | 検証 |\n|---|---|---|---|---|---|---|---|\n${issues
    .map((i) => `| ${i.id} | ${SEVERITY_LABEL[i.severity]} | ${i.kind} | ${fmtLoc(i.location)} | ${normText(i.claim).replace(/\|/g, '/').slice(0, 80)} | ${STATUS_LABEL[i.status]} | ${{ claude: 'Claude', judge: '判定役', user: 'ユーザー' }[i.decided_by] ?? '—'} | ${verifiedIssue(i) ? '✓' : ''} |`)
    .join('\n')}`);
  const cases = Object.values(state.judge_cases);
  if (cases.length) {
    md.push(`## 判定役による裁定\n${cases.map((c) => `- ${c.id}（${c.issue}、理由: ${c.reason}）: ${c.resolved ? { upheld: '指摘を支持', rejected: '却下を支持', split: '割れたため保留', user: 'ユーザーが決定' }[c.resolved.result] : '未確定'}${Object.entries(c.verdicts).map(([j, v]) => `\n  - ${j}: ${v.winner === 'undecidable' ? '判断不能' : c.mapping[v.winner] === 'issue' ? '指摘側' : '却下側'}（重要度 ${SEVERITY_LABEL[v.severity]}）${truncate(normText(v.reasoning), 200)}`).join('')}`).join('\n')}`);
  }
  if (state.cycle_extra.length) md.push(`## 指摘以外の変更（宣言済み）\n${state.cycle_extra.map((x) => `- ラウンド ${x.round}: ${x.file} — ${x.reason}`).join('\n')}`);
  if (chk.errors.length) md.push(`## 反映確認の警告\n${chk.errors.map((e) => `- ${e}`).join('\n')}`);
  md.push(`## 最終確認（議論を知らない Codex）\n${state.final_check?.skipped ? '省略（設定により）' : `- 判定: ${state.final_check?.verdict}\n- ${state.final_check?.summary ?? ''}\n- 新しい指摘: ${state.final_check?.new_ids?.join(', ') || 'なし'}${finalChanged ? '\n- 注意: 最終確認の後に成果物が変更されています' : ''}`}`);
  const openQs = state.questions.filter((q) => !q.answer);
  if (unresolved.length || openQs.length) {
    md.push(`## 未解決・要判断\n${unresolved.map((i) => `- ${i.id}（${STATUS_LABEL[i.status]}、重要度 ${SEVERITY_LABEL[i.severity]}）${normText(i.claim).slice(0, 200)}`).join('\n')}${openQs.length ? `\n${openQs.map((q) => `- ${q.id}（質問）${q.text}`).join('\n')}` : ''}`);
  }
  const backlog = issues.filter((i) => i.status === 'backlog');
  if (backlog.length) md.push(`## 後回しリスト\n${backlog.map((i) => `- ${i.id}（${SEVERITY_LABEL[i.severity]}${i.scope === 'out' ? '、範囲外' : ''}）${normText(i.claim).slice(0, 200)}`).join('\n')}`);
  if (Object.keys(state.verifies).length) {
    md.push(`## 検証ログ\n${Object.values(state.verifies).map((v) => `- ${v.id}${v.issue ? `（${v.issue}）` : ''}${v.auto ? '（自動）' : ''}: \`${v.cmd}\` → ${v.exit}${integrity.includes(v.id) ? ' ⚠ 記録が改変されています' : ''}`).join('\n')}`);
  }
  md.push(`## 初稿からの変更\n${diff ? `${'```diff'}\n${truncate(diff, 20000)}\n${'```'}\n全文: diff.patch` : '変更なし'}`);
  md.push(`## コスト\n- Codex: ${codexUse.calls} 回、${Math.round(codexUse.duration_ms / 1000)} 秒、入力 ${codexUse.input.toLocaleString()} / 出力 ${codexUse.output.toLocaleString()} トークン${claudeUse.calls ? `\n- Claude CLI 判定役: ${claudeUse.calls} 回、${Math.round(claudeUse.duration_ms / 1000)} 秒${claudeUse.cost_usd ? `、$${claudeUse.cost_usd.toFixed(3)}` : ''}` : ''}${state.settings.judges.includes('claude') && state.settings.claudeJudgeMode === 'agent' ? '\n- Claude 判定役（サブエージェント）の消費量はこのレポートに含まれません' : ''}`);
  const reportFile = path.join(dir, 'report.md');
  writeText(reportFile, `${md.join('\n\n')}\n`);
  const result = {
    id: state.id, topic: state.topic, terminal: state.terminal, rounds: state.round,
    final_artifacts: state.artifacts.map((a) => path.resolve(state.cwd, a)),
    counts: Object.fromEntries(Object.keys(STATUS_LABEL).map((k) => [k, issues.filter((i) => i.status === k).length])),
    unresolved: unresolved.map((i) => ({ id: i.id, status: i.status, severity: i.severity, claim: i.claim })),
    open_questions: openQs,
    backlog: backlog.map((i) => ({ id: i.id, severity: i.severity, claim: i.claim })),
    apply_warnings: chk.errors, verify_integrity_errors: integrity,
    final_check: state.final_check, final_changed_after_check: !!finalChanged,
    usage: { codex: codexUse, claude_cli: claudeUse },
    report: reportFile, diff: path.join(dir, 'diff.patch'),
  };
  writeJSON(path.join(dir, 'result.json'), result);
  state.reported_hash = artifactsHash(state);
  logLine(ctx, 'レポートを作成');
  saveState(ctx);
  out({ ok: true, ...result });
}

async function cmdStatus(opts) {
  const ctx = loadDebate(opts);
  const { state } = ctx;
  refreshPhase(state);
  const issues = Object.values(state.issues);
  out({
    ok: true, id: state.id, dir: ctx.dir, topic: state.topic, phase: state.phase, round: state.round, max_rounds: state.settings.rounds,
    terminal: state.terminal, settings: state.settings, artifacts: state.artifacts, context_file: path.join(ctx.dir, 'context.md'),
    counts: Object.fromEntries(Object.keys(STATUS_LABEL).map((k) => [k, issues.filter((i) => i.status === k).length])),
    open: issues.filter((i) => i.status === 'open').map(issueView),
    escalated: issues.filter((i) => i.status === 'escalated').map((i) => i.id),
    held: issues.filter((i) => i.status === 'held').map(issueView),
    final_open: issues.filter((i) => i.status === 'final_open').map(issueView),
    unanswered_questions: state.questions.filter((q) => !q.answer),
    pending_judge_cases: Object.values(state.judge_cases).filter((c) => !c.resolved).map((c) => ({ case: c.id, issue: c.issue, have: Object.keys(c.verdicts), packet: c.packet })),
    next_action: state.next_action,
  });
}

async function cmdList() {
  const rows = indexRead().map((e) => {
    const sf = path.join(e.dir, 'state.json');
    if (!exists(sf)) return { ...e, phase: '（削除済み）' };
    const st = readJSON(sf);
    return { id: e.id, created_at: e.created_at, cwd: e.cwd, topic: e.topic, phase: st.phase, round: st.round, terminal: st.terminal?.status ?? null };
  });
  out({ ok: true, debates: rows.reverse() });
}

async function cmdClean(opts) {
  const days = opts['older-than-days'] ? parseInt(opts['older-than-days'], 10) : 30;
  if (!(days >= 0)) fail('--older-than-days は 0 以上');
  const limit = Date.now() - days * 86400000;
  const idx = indexRead();
  const targets = idx.filter((e) => Date.parse(e.created_at) < limit && exists(path.join(e.dir, 'state.json')) && e.dir.includes(`${path.sep}debates${path.sep}`));
  if (!opts.yes) {
    out({ ok: true, dry_run: true, would_delete: targets.map((e) => ({ id: e.id, dir: e.dir, topic: e.topic })), note: '実際に削除するにはユーザーの確認を得てから --yes を付ける' });
    return;
  }
  for (const e of targets) fs.rmSync(e.dir, { recursive: true, force: true });
  const keep = idx.filter((e) => !targets.includes(e));
  writeText(INDEX_FILE, keep.map((e) => JSON.stringify(e)).join('\n') + (keep.length ? '\n' : ''));
  out({ ok: true, deleted: targets.map((e) => e.id) });
}

async function cmdDoctor(opts) {
  const cfg = loadConfig(process.cwd());
  const checks = [];
  const add = (name, ok, detail, fix) => checks.push({ name, ok, detail, ...(fix ? { fix } : {}) });
  const v = spawnSync(cfg.codexBin, ['--version'], { encoding: 'utf8' });
  const ver = (v.stdout || '').match(/(\d+\.\d+\.\d+)/)?.[1];
  add('codex-version', !!ver && compareVersions(ver, cfg.minCodexVersion) >= 0, ver ? `codex-cli ${ver}（必要: ${cfg.minCodexVersion} 以上）` : 'codex が見つかりません', 'brew upgrade --cask codex');
  const l = spawnSync(cfg.codexBin, ['login', 'status'], { encoding: 'utf8', timeout: 30000 });
  add('codex-login', l.status === 0, truncate(`${l.stdout || ''}${l.stderr || ''}`.trim(), 200), 'codex login');
  let stale = findStaleCodexProcesses(cfg.codexBin);
  if (opts.fix && stale.length) {
    for (const p of stale) { try { process.kill(p.pid, 'SIGTERM'); } catch { /* 既に終了 */ } }
    stale = stale.map((p) => ({ ...p, killed: true }));
  }
  add('stale-processes', !stale.length || stale.every((p) => p.killed), stale.length ? stale : '更新前から動いている Codex の常駐プロセスはありません', 'doctor --fix（ユーザーの確認後）');
  for (const f of fs.readdirSync(SCHEMA_DIR)) {
    try { readJSON(path.join(SCHEMA_DIR, f)); add(`schema:${f}`, true, 'ok'); } catch (e) { add(`schema:${f}`, false, e.message); }
  }
  for (const bin of ['git', 'diff']) {
    const r = spawnSync('/bin/sh', ['-c', `command -v ${bin}`], { encoding: 'utf8' });
    add(`bin:${bin}`, r.status === 0, (r.stdout || '').trim() || '見つかりません');
  }
  if (cfg.judges.includes('claude')) {
    if (cfg.claudeJudgeMode === 'cli') {
      const bin = findClaudeBin(cfg);
      const r = spawnSync(bin, ['--version'], { encoding: 'utf8' });
      add('claude-judge', r.status === 0, `cli: ${bin} ${(r.stdout || '').trim()}`);
    } else add('claude-judge', true, 'agent モード（Claude 本体がサブエージェントで判定させる）');
  }
  for (const f of [USER_CONFIG, path.join(process.cwd(), '.claude', 'debate.json')]) {
    if (!exists(f)) continue;
    try { readJSON(f); add(`config:${f}`, true, 'ok'); } catch (e) { add(`config:${f}`, false, e.message); }
  }
  out({ ok: checks.every((c) => c.ok), checks });
}

// ───────────────────────── エントリポイント ─────────────────────────

const COMMANDS = {
  init: cmdInit, blind: cmdBlind, round: cmdRound, verify: cmdVerify, respond: cmdRespond, judge: cmdJudge,
  decide: cmdDecide, answer: cmdAnswer, 'declare-change': cmdDeclareChange, 'add-artifact': cmdAddArtifact,
  'final-check': cmdFinalCheck, report: cmdReport, status: cmdStatus, list: cmdList, clean: cmdClean, doctor: cmdDoctor,
};

async function main() {
  const opts = parseArgs(process.argv.slice(2));
  const cmd = opts._[0];
  const fn = COMMANDS[cmd];
  if (!fn) fail(`使い方: debate.mjs <${Object.keys(COMMANDS).join('|')}> [options]`);
  await fn(opts);
}

main().catch((e) => {
  if (e instanceof UserError) out({ ok: false, error: e.message, ...e.extra });
  else out({ ok: false, error: `内部エラー: ${e.stack || e}` });
  process.exitCode = 1;
});
`````

### `~/.claude/skills/debate/schemas/blind.json`

期待される行数: 10

`````json
{
  "type": "object",
  "additionalProperties": false,
  "required": ["solution", "key_points", "assumptions"],
  "properties": {
    "solution": { "type": "string" },
    "key_points": { "type": "array", "items": { "type": "string" } },
    "assumptions": { "type": "array", "items": { "type": "string" } }
  }
}
`````

### `~/.claude/skills/debate/schemas/critique.json`

期待される行数: 52

`````json
{
  "type": "object",
  "additionalProperties": false,
  "required": ["verdict", "summary", "issues", "resolved_ids", "questions"],
  "properties": {
    "verdict": { "type": "string", "enum": ["AGREE", "REVISE"] },
    "summary": { "type": "string" },
    "issues": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["ref_id", "severity", "kind", "scope", "location", "claim", "impact", "evidence", "suggestion", "verification"],
        "properties": {
          "ref_id": { "type": ["string", "null"] },
          "severity": { "type": "string", "enum": ["high", "medium", "low"] },
          "kind": { "type": "string", "enum": ["bug", "factual", "logic", "missing", "design", "clarity", "style", "other"] },
          "scope": { "type": "string", "enum": ["in", "out"] },
          "location": {
            "type": "object",
            "additionalProperties": false,
            "required": ["file", "line_start", "line_end", "section"],
            "properties": {
              "file": { "type": ["string", "null"] },
              "line_start": { "type": ["integer", "null"] },
              "line_end": { "type": ["integer", "null"] },
              "section": { "type": ["string", "null"] }
            }
          },
          "claim": { "type": "string" },
          "impact": { "type": "string" },
          "evidence": { "type": "string" },
          "suggestion": { "type": "string" },
          "verification": { "type": ["string", "null"] }
        }
      }
    },
    "resolved_ids": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["id", "reason"],
        "properties": {
          "id": { "type": "string" },
          "reason": { "type": "string" }
        }
      }
    },
    "questions": { "type": "array", "items": { "type": "string" } }
  }
}
`````

### `~/.claude/skills/debate/schemas/judge.json`

期待される行数: 12

`````json
{
  "type": "object",
  "additionalProperties": false,
  "required": ["case_nonce", "winner", "severity", "confidence", "reasoning"],
  "properties": {
    "case_nonce": { "type": "string" },
    "winner": { "type": "string", "enum": ["X", "Y", "undecidable"] },
    "severity": { "type": "string", "enum": ["high", "medium", "low"] },
    "confidence": { "type": "number" },
    "reasoning": { "type": "string" }
  }
}
`````

### `~/.claude/skills/debate/schemas/response.json`

期待される行数: 39

`````json
{
  "$comment": "Claude が rounds/rN/response.json に書く回答の形式（debate.mjs respond が検証する）",
  "type": "object",
  "required": ["decisions"],
  "properties": {
    "decisions": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id", "decision", "rationale"],
        "properties": {
          "id": { "type": "string" },
          "decision": { "enum": ["accept", "partial", "reject"] },
          "reason_type": { "enum": ["incorrect", "out_of_scope", "tradeoff", "duplicate", "already_handled", null] },
          "duplicate_of": { "type": ["string", "null"] },
          "rationale": { "type": "string" },
          "evidence": {
            "type": "array",
            "items": {
              "oneOf": [
                { "type": "object", "required": ["type", "file", "line_start", "line_end", "text"], "properties": { "type": { "const": "quote" }, "file": { "type": "string" }, "line_start": { "type": "integer" }, "line_end": { "type": "integer" }, "text": { "type": "string" }, "version": { "type": "string" } } },
                { "type": "object", "required": ["type", "id"], "properties": { "type": { "const": "verify" }, "id": { "type": "string" } } },
                { "type": "object", "required": ["type", "url", "text"], "properties": { "type": { "const": "url" }, "url": { "type": "string" }, "text": { "type": "string" } } }
              ]
            }
          },
          "applied_at": {
            "type": "array",
            "items": { "type": "object", "required": ["file"], "properties": { "file": { "type": "string" }, "line_start": { "type": ["integer", "null"] }, "line_end": { "type": ["integer", "null"] } } }
          }
        }
      }
    },
    "extra_changes": {
      "type": "array",
      "items": { "type": "object", "required": ["file", "reason"], "properties": { "file": { "type": "string" }, "reason": { "type": "string" } } }
    }
  }
}
`````

### `~/.claude/debate.json`

期待される行数: 23

`````json
{
  "rounds": 3,
  "effort": null,
  "model": null,
  "timeoutSec": 900,
  "retries": 1,
  "logDir": ".claude/debates",
  "blockingSeverities": ["high"],
  "escalation": { "rejectRepeat": 2, "rejectRateThreshold": 0.6, "minDecisionsForRate": 3 },
  "judges": ["claude", "codex"],
  "claudeJudgeMode": "agent",
  "finalCheck": true,
  "minCodexVersion": "0.162.0",
  "verifyTimeoutSec": 300,
  "testCommands": [],
  "applyTolerance": 1,
  "secretPatterns": [],
  "secretAllow": [],
  "presets": {
    "quick": { "rounds": 1, "effort": "low", "judges": [], "finalCheck": false },
    "deep": { "rounds": 5, "effort": "xhigh" }
  }
}
`````
