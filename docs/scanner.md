# 台帳スキャナ 設計

2026-11-05 · 対象: Claude Code のみ · 関連: [spec.md](spec.md)

台帳スキャナは、Claude Code が読み込むハーネスを部品（コンポーネント）単位で列挙し、`inventory.json` に書き出すコマンド（`refit scan`）。

最も重要な設計判断は、**部品の単位を「使用率を観測できる単位」に合わせること**。台帳の ID が hooks のイベントと突き合わせられなければ、後段の使用率ダッシュボードが成立しない。

## スキャン対象

ユーザー階層は `~/.claude`（`CLAUDE_CONFIG_DIR` が設定されていればそちら）、プロジェクト階層はリポジトリのルートを基準にする。

| 種別 (kind) | ユーザー | プロジェクト | 部品の単位 | 読み込まれ方 |
| --- | --- | --- | --- | --- |
| `instructions` | `~/.claude/CLAUDE.md` | `CLAUDE.md`, `.claude/CLAUDE.md`, `CLAUDE.local.md`, `AGENTS.md`, サブディレクトリの `CLAUDE.md` | 1ファイル（`##` 見出しを子要素として保持） | 常時。サブディレクトリ分は該当ディレクトリに触れたとき |
| `rule` | `~/.claude/rules/*.md` | `.claude/rules/**/*.md` | 1ファイル | `paths:` なしは常時、ありは一致するファイルを読んだとき |
| `skill` | `~/.claude/skills/<name>/SKILL.md` | `.claude/skills/<name>/SKILL.md` | 1スキル | description は常時、本文は起動時 |
| `command` | `~/.claude/commands/*.md` | `.claude/commands/*.md` | 1ファイル | ユーザーが `/name` で起動 |
| `agent` | `~/.claude/agents/*.md` | `.claude/agents/*.md` | 1エージェント | description は常時、本文はサブエージェント起動時 |
| `hook` | `~/.claude/settings.json` の `hooks` | `.claude/settings.json`, `.claude/settings.local.json` の `hooks` | 1つの「イベント × matcher × コマンド」 | 該当イベント発火時 |
| `mcp_server` | `~/.claude.json` の `mcpServers` と `projects.<path>.mcpServers` | `.mcp.json` | 1サーバー | ツール定義は常時または遅延読み込み、呼び出しは都度 |
| `output_style` | `~/.claude/output-styles/*.md` | `.claude/output-styles/*.md` | 1ファイル | `outputStyle` で選択中のものだけ常時 |
| `workflow` | `~/.claude/workflows/*.js` | `.claude/workflows/*.js` | 1ファイル | `/name` で起動 |
| `plugin` | `~/.claude/plugins/` と `enabledPlugins` | `.claude/settings.json` の `enabledPlugins` | 1プラグイン（中の skill・agent・hook・MCP を子として展開） | 有効化されている間 |
| `permission` | 各 settings の `permissions` | 同左 | 1ルール | 常時（台帳には載せるが、MVPでは使用率を測らない） |

管理設定（`managed-settings.json` など）は読み取りのみ行い、`scope: managed` として記録する。refit からは変更しない。

**対象外**：自動メモリ（`~/.claude/projects/<project>/memory/`）とサブエージェントのメモリ。どちらもハーネスというより蓄積データで、内容が日々変わるため。ただし、存在とサイズだけは記録する。

## プロジェクトの列挙

既定では、カレントディレクトリのプロジェクトとユーザー階層だけを読む。

`--all-projects` を付けると、`~/.claude.json` の `projects` キーに記録されたパス（Claude Code を使ったことのあるプロジェクトの一覧）を順に読む。ハーネスがどのプロジェクトにどう散らばっているかを一度に把握できる。

## 部品の記録項目

各部品は次の項目を持つ。

| 項目 | 例 | 用途 |
| --- | --- | --- |
| `id` | `skill:user:pdf-reading` | 台帳全体で一意。使用イベントとの突き合わせに使う |
| `kind` | `skill` | 上の表の種別 |
| `scope` | `user` / `project` / `local` / `managed` / `plugin` | どの階層にあるか |
| `name` | `pdf-reading` | 表示名 |
| `source` | `~/.claude/skills/pdf-reading/SKILL.md` | 実ファイルの場所（行範囲つき） |
| `plugin` | `pstack@marketplace` | プラグイン由来の場合の出どころ |
| `trigger` | description、`paths:` の glob、hook の matcher | どんな条件で読み込まれ、起動されるか |
| `load` | `always` / `on_demand` / `path_glob` / `event` / `user_invoked` | 読み込まれ方 |
| `context_cost` | `{ always: 120, on_load: 2400 }`（トークン概算） | 常時コンテキストを食う量と、起動時に増える量 |
| `content_hash` | `sha256:…` | 変更の検出。ablation 結果がどの版に対するものかの紐付け |
| `mtime` | `2026-10-30T…` | 最終更新日 |
| `status` | `active` / `shadowed` / `disabled` | 同名部品に上書きされている、無効化されている、など |
| `shadowed_by` | `skill:project:pdf-reading` | 上書きしている部品の ID |

`context_cost` は判定（とくに CUT）に直結する。常時読み込まれる部品は、使われなくてもコストがかかるため、「常時コスト大・使用率ゼロ」が最初に目立つようにする。トークン数は文字数からの概算でよい（精度より比較できることを優先する）。

`status: shadowed` も重要。スキルとコマンドの同名、ユーザーとプロジェクトの同名などで、置いてあるのに効いていない部品は、ablation 以前に検出できる。

### ID の付け方

```
<kind>:<scope>:<name>[#<child>]
```

| 例 | 意味 |
| --- | --- |
| `instructions:user:CLAUDE.md` | ユーザーの CLAUDE.md |
| `instructions:project:CLAUDE.md#testing` | プロジェクト CLAUDE.md の「Testing」節 |
| `rule:project:api-conventions` | `.claude/rules/api-conventions.md` |
| `skill:plugin:pstack:setup-pstack` | プラグイン pstack のスキル |
| `hook:user:PostToolUse:Edit|Write#0` | ユーザー設定の PostToolUse フックの1番目 |
| `mcp_server:user:memory` | ユーザーの MCP サーバー memory |

プロジェクト階層の ID には、どのプロジェクトかを示す識別子（リポジトリルートのパスのハッシュ）を別項目 `project` として持たせる。ID 自体は短く保つ。

## 使用率との結合キー

使用率コレクタ（MVP の2番目）は hooks で使用イベントを受け取る。台帳の各部品が、どのイベントのどのフィールドで「使われた」と判定できるかを先に決めておく。

| 種別 | イベント | 結合キー | 確度 |
| --- | --- | --- | --- |
| `instructions`, `rule` | `InstructionsLoaded` | 読み込まれたファイルのパス | 要確認（パスのフィールド名がドキュメントで未確認） |
| `skill`（モデルが起動） | `PreToolUse`（`tool_name` が `Skill`） | `tool_input` 内のスキル名 | 要確認（フィールド名が未確認） |
| `skill`, `command`（ユーザーが起動） | `UserPromptExpansion` | matcher のコマンド名 | 確認済み |
| `agent` | `SubagentStart` | `agent_type` | 確認済み |
| `mcp_server` | `PreToolUse` | `tool_name` の `mcp__<server>__<tool>`（プラグイン由来は `mcp__plugin_<plugin>_<server>__<tool>`） | 確認済み |
| `hook` | なし | フック自身の発火は hooks では観測できない | MVP では測らない |
| `output_style` | `SessionStart` | 選択中の `outputStyle` | 常時なのでセッション数として数える |
| `permission` | `PermissionRequest`, `PermissionDenied` | ルール | MVP では測らない |

**読み込まれた（loaded）と使われた（invoked）は別に数える。** たとえば rule は「読み込まれた」しか観測できないが、skill は description が常時読み込まれたうえで「起動された」を観測できる。ダッシュボードではこの2つを混ぜない。

CLAUDE.md の節単位の使用率は観測できない（`InstructionsLoaded` はファイル単位でしか発火しない）。節単位で判定したい場合は、ablation でしか測れない。spec.md では「1ルール」を最小単位としていたが、台帳は節単位まで持ち、使用率はファイル単位で集計する形に分ける。

### 過去分の取り込み（バックフィル）

hooks を仕込んだ日からしかデータが溜まらないと、ダッシュボードが空のまま数週間過ぎる。そこで、`~/.claude/projects/<project>/<session>.jsonl`（トランスクリプト、既定で30日分残る）から過去の使用を復元する `refit backfill` を用意する。

トランスクリプトには `tool_use` ブロック（ツール名と入力）と、セッション開始時のスキル一覧などが記録されている。ここから MCP、スキル、サブエージェントの使用回数が取れる。

ただし、トランスクリプトの形式は公開仕様ではなく、変わりうる。バックフィルは補助と位置づけ、継続的な計測は hooks を正とする。

## 出力と CLI

```
refit scan                  # カレントプロジェクト + ユーザー階層をスキャンし、表で表示
refit scan --json           # inventory.json を標準出力へ
refit scan --all-projects   # ~/.claude.json に記録された全プロジェクトを対象にする
refit diff                  # 前回スキャンとの差分（追加・削除・変更・上書き状態の変化）
```

出力先は `~/.refit/inventory.json`。スキャンのたびにスナップショットを `~/.refit/snapshots/<timestamp>.json` に残し、`refit diff` で比較する。モデルのリリース前後でハーネスがどう変わったかを追えるようにするため。

`inventory.json` は「観測した現状」であり、spec.md でいう宣言的マニフェスト（あるべき姿）とは別物にする。マニフェストが必要になるのは移行（後続フェーズ）なので、MVP ではマニフェストの形式（YAML / Nix など）を決めなくて済む。

## プライバシーと安全

- `~/.claude/.credentials.json` は読まない
- settings の `env`、MCP の `env` と `headers` は、キー名だけ記録し値は捨てる
- トランスクリプトから取るのは部品 ID と回数・日時だけ。プロンプトやツールの出力は保存しない
- すべてローカルで完結させ、外部に送信しない
- 書き込むのは `~/.refit/` だけ。Claude Code の設定ファイルは一切変更しない

## 実装方針（案）

- **言語**：TypeScript（Node.js）。Claude Code 本体と同じ実行環境で、hooks のスクリプトもそのまま書ける
- **配布**：`npx refit` で動く単一パッケージ
- **構成**：種別ごとに独立したスキャナ（`scanners/skill.ts` など）を置き、共通の `Component` 型を返す。Claude Code の仕様変更が入っても、影響を受けたスキャナだけ直せばよい
- **テスト**：架空の `~/.claude` と `.claude` を持つフィクスチャディレクトリを用意し、スキャン結果のスナップショットテストで固定する

## 未確認・未決事項

- [ ] `InstructionsLoaded` の入力に含まれるファイルパスのフィールド名（記録用フックを仕込んで実機で確認する）
- [ ] `Skill` ツールの `tool_input` でスキル名が入るフィールド名（同上）
- [ ] インストール済みプラグインの一覧を `claude plugin list` の出力から取れるか、`~/.claude/plugins/` を直接読む必要があるか
- [ ] サブディレクトリの CLAUDE.md をどこまで深く探すか（node_modules などの除外ルール）
- [ ] トークン概算を文字数ベースで済ませるか、トークナイザーを使うか
- [ ] 実装言語を TypeScript で確定してよいか

## 参考

- [Settings files and precedence](https://code.claude.com/docs/en/settings)
- [Claude directory reference](https://code.claude.com/docs/en/claude-directory)
- [Hooks reference](https://code.claude.com/docs/en/hooks)
- [Plugin manifest reference](https://code.claude.com/docs/en/plugins-reference)
