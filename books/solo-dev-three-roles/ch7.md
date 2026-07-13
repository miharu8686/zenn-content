---
title: "付録: コピペ可能な設定一式"
free: false
---

# 第7章 付録——コピペで導入できる設定一式

## 7.0 この付録の使い方

ここまでの6章で見てきた仕組みは、どれも特別な部品でできていません。gitのフック、数枚のMarkdownファイル、Pythonスクリプト、そしてClaude Codeにもとから備わっている機能。この付録は、それらを実際に手元へ移せる形——コピー&ペーストして、自分の環境に合わせて数箇所を書き換えれば動く形——で並べたものです。

読む前に、二つだけ約束事があります。

第一に、この付録は**依存の順に3層で並んでいます**。第1層(規律)は依存ゼロで、単体でそのまま効きます。第2層(委譲)は第1層の上に乗ります。第3層(資産化)は、第4章で見たSTATE運用がすでに回っていることを前提にします。上から順に入れれば、前提が満たされないまま次へ進むことはありません。逆に言えば、**全部を入れる必要はありません**。第1層のフック3本を入れるだけでも、第1章で見た事故の何割かは即座に止まります。まず第1層だけ試し、効き目を確かめてから下の層へ進む、という読み方を勧めます。

第二に、この付録の加工版には、一つの流儀を通してあります。**書き換える場所を、隠さずに宣言する**という流儀です。

第4章で、「書式は美学ではなく、機械との契約である」と書きました。散文の中の一行が、実は自動転記スクリプトのAPIスキーマとして機能している——その結合を隠すのではなく、宣言することで運用は安定する、と。この付録は、その主張を付録自身で体現します。環境依存の値を外部ファイルへ追い出して見えなくするのではなく、コードや設定の先頭に「ここを書き換える」と露出させたまま置き、なぜそこを書き換えるのかを一行添えます。隠された結合は事故を生み、宣言された結合は運用を支える。付録もまた、その原則の下にあります。

具体的には、書き換えるべき箇所はすべて `<...>` という山括弧のプレースホルダで書いてあります。導入したら、`<` で全文を検索(grep)してください。ヒットした箇所が、あなたが自分の環境に合わせて埋めるべき場所の全リストです。埋め忘れれば `<` が残るので、検索一発で漏れが分かります。各素材の冒頭には「**書き換える箇所**」の一覧を置き、その一つずつに理由を添えました。

なお、以下の設定ファイルのパスはすべて、Claude Codeの設定ディレクトリ(本書では `~/.claude/` と表記します。実体はOSによって異なります)を起点にした相対的な位置です。

---

## 7.1 第1層：規律（依存ゼロ・単体で効く）

この層は、第1章で最も頻度の高かった失敗——git規律違反と推測進行——に、直接の壁を立てます。他のどの設定にも依存しません。入れた瞬間から、該当するコマンドは実行前に機械的に拒否され、セッションの冒頭には現在の状態が自動で注入されます。

### CLAUDE.md（共通規律）

**書き換える箇所**: なし（汎用の開発規律のみ。ただし表題に個人名などを入れている場合は外す）
**本文の解説**: 第2章

すべてのリポジトリに共通する開発規律を、`~/.claude/CLAUDE.md` に置きます。これはプロンプトではなく、全セッションの背骨に常時読み込まれる憲法です。固有のパスも秘匿情報も含まないので、そのまま使えます。

```markdown
# 共通開発規律（全リポジトリ共通）
リポジトリ固有のルールは各リポジトリ直下の CLAUDE.md にある。矛盾したらリポジトリ側が優先。

## 着手前（必須）
- `git log --oneline -5` と `git status` で実体を確認してから作業する。
- 渡された指示・STATEファイルとリポジトリ実体が食い違ったら、作業を止めて報告する（実体が正）。

## git規律
- add は対象ファイル指定のみ。`git add -A` / `git add .` 禁止。
- コミットメッセージはファイル経由（`git commit -F <file>`）。heredoc 禁止。
- push は指示されたブランチへ。main へのマージは行わない（人間の検収後）。
- push 後に `git branch -vv` で追跡状態を確認し、報告に含める（push忘れ防止）。
（上記のうち git add -A/./--all 禁止・heredocコミット禁止・main直push禁止は
settings.json のhooksで機械的に強制されている。コマンドが実行前に拒否される場合はこれが原因）

## 変更の進め方
- 指示された「触ってよい範囲」の外は変更しない。目的達成に範囲外の変更が必要なら、理由を報告してから行う。
- 絡み合う変更（配置・表示・状態・依存）は1変更ずつ。既存挙動を変えない read-only 追加を優先。
- 原因不明の不具合は、修正候補を試す前に「正常に動く参照」との対照実験で層を切り分ける。
- 使い捨て検証は spike 用ディレクトリで行い、本番リポジトリに混ぜない。

## コスト・プロセス安全
- API課金が走る処理は、実行前に概算を報告して承認を待つ。
- プロセス停止は Name と CommandLine の両方で厳密一致させる。緩いフィルタ禁止。
- 稼働中プロセスは旧コードを保持する。コード反映は再起動で行う。

## 報告
- 端的な日本語・絵文字なし。
- 「ビルド成功」と「表示・動作が正しい」は別物として報告する。最終目視は人間が行う。
- 完了報告には、変更ファイル一覧・コミットハッシュ・push先ブランチを含める。

## サブエージェント運用
書き込み権限を持つのはメインセッションのみ。サブエージェントは全員読み取り専用の専門職。該当局面で積極的に委譲する。
標準フロー:
1. セッション開始/STATE受領 → ground-truth-check
2. 調査 → explorer（検索）／ contrast-debugger（原因不明の不具合。仮説修正より先に必ず）
3. 実装（メインセッション自身が行う） → test-runner → code-reviewer
4. コミット後 → pre-merge-check
```

### settings.json PreToolUse（git規律の機械強制・3本）

**書き換える箇所**: なし（汎用の強制パターンのみ。環境が割れる情報を含まない）
**本文の解説**: 第3章

第3章で「体制の物理法則」と呼んだフックです。`~/.claude/settings.json` の `hooks.PreToolUse` に置くと、Bash / PowerShell 経由のコマンドが実行される**前**に発火し、条件に当たれば `exit 2` で止めます。事後に謝る仕組みではなく、事前に不可能にする仕組みです。

三本の内訳は、`git add -A` / `--all` / `.` の拒否、heredoc を使ったコミットの拒否、main への直接 push の拒否。いずれも拒否メッセージが「次の正しい行動」を指示している点に注目してください（第3章）。壁ではなく通路になっています。

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash|PowerShell",
        "hooks": [
          {
            "type": "command",
            "command": "cmd=$(cat | jq -r '.tool_input.command // \"\"'); if echo \"$cmd\" | grep -qE 'git add +(-A|--all|\\.)([^A-Za-z0-9_./-]|$)'; then echo 'git add -A/./--all は禁止。対象ファイルを明示指定してください。' >&2; exit 2; fi; exit 0"
          }
        ]
      },
      {
        "matcher": "Bash|PowerShell",
        "hooks": [
          {
            "type": "command",
            "command": "cmd=$(cat | jq -r '.tool_input.command // \"\"'); if echo \"$cmd\" | grep -q 'git commit' && echo \"$cmd\" | grep -qE '<<'; then echo 'コミットメッセージはheredocでなく -F <file> で渡してください。' >&2; exit 2; fi; exit 0"
          }
        ]
      },
      {
        "matcher": "Bash|PowerShell",
        "hooks": [
          {
            "type": "command",
            "command": "cmd=$(cat | jq -r '.tool_input.command // \"\"'); if echo \"$cmd\" | grep -qE 'git push[^|&]*\\bmain\\b'; then echo 'mainへの直接pushは禁止。featブランチへpushし、人間の検収後にマージしてください。' >&2; exit 2; fi; exit 0"
          }
        ]
      }
    ]
  }
}
```

`matcher` が `Bash|PowerShell` の両方を指している点が肝です。第3章で見たとおり、初期の版は `Bash` だけを見ており、PowerShell 経由のコマンドが素通りしていました。禁止したはずの操作の半分が禁止されていなかったのです。あなたが使うシェルが何であれ、両方を書いておくのが安全です。

（このフックは `jq` を使います。手元に無ければ導入してください。また `grep` の正規表現は、`git add .\foo.txt` のような環境依存の慣用表現を偽陽性で弾くことがあります——第3章で実際に起きた盲点です。導入後、自分が普段使うadd記法が弾かれないか、無害なコマンドで一度確かめておくと安心です。）

### settings.json SessionStart（git状態の自動注入）

**書き換える箇所**: なし（汎用の git コマンドのみ）
**本文の解説**: 第3章

同じフックの仕組みを、止めるためでなく**配る**ために使います。第3章の後半で見た、知覚を毎回強制的に配る装置です。セッションが始まるたびに直近3コミット・現在ブランチ・作業ツリーの汚れ具合を会話の冒頭へ注入し、実装役が最初の一文字を読む前に「現在」を食べさせます。`hooks.SessionStart` に追加します。

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo '=== git status (auto-injected) ==='; git log --oneline -3 2>/dev/null; echo '---'; git branch --show-current 2>/dev/null; echo '---'; git status --short 2>/dev/null; exit 0"
          }
        ]
      }
    ]
  }
}
```

禁止フックと注入フックは、同じ仕組みの表裏です。前者は悪い行動の確率をゼロに寄せ、後者は正しい知覚の確率を1に寄せる。どちらも意志の問題を配線の問題へ翻訳しています。

---

## 7.2 第2層：委譲（第1層に乗る）

この層は、第2章の三役体制——読み取り専用のサブエージェントと、拒否権つきのインターフェース——を実装します。第1層の規律が下敷きにあることを前提にします。

### settings.json PreToolUse G4（秘匿ファイル遮断）

**書き換える箇所**:
- `<secrets>\.enc` … あなたの環境の暗号化済み秘匿ファイル名（正規表現）
- `<userdata>\.json` … 保護したい実ユーザーデータのファイル名（正規表現）

**本文の解説**: 第3章

7.1 の PreToolUse に、四本目として追加します。秘匿ファイルへのアクセスを、コマンド実行前に遮断するフックです。第3章で述べたとおり、これは事故を待たずに置かれた予防的な配線です。実ファイル名はそのまま書くと環境が割れるため、プレースホルダにしてあります。自分が守りたいファイル名の正規表現に置き換えてください（`.env` は汎用の例としてそのまま残してあります。ただし `.env.example` のようなテンプレートまで巻き込まないよう、パターンの作り込みには注意——これも第3章で見た過剰マッチの盲点です）。

```json
{
  "matcher": "Bash|PowerShell",
  "hooks": [
    {
      "type": "command",
      "command": "cmd=$(cat | jq -r '.tool_input.command // \"\"'); if echo \"$cmd\" | grep -qiE '(<secrets>\\.enc|\\.env(\\.local)?(\\s|$|[\"'\\''\\)])|<userdata>\\.json)'; then echo '秘匿ファイル/実ユーザーデータへのアクセスは禁止されています。' >&2; exit 2; fi; exit 0"
    }
  ]
}
```

このフックの `matcher` は Bash / PowerShell だけを見ています。つまり Read や Grep といったツール経由の読み取りには掛かりません（第3章の 2-d残）。ここはモデルの自制に依存する残余の盲点です。塞げない穴を、塞げないと知った上で残している——盲点の記録として、あえて明記しておきます。

### agents/（サブエージェント6体）

**書き換える箇所**: なし（汎用定義。固有パス・プロジェクト名を含まない）
**本文の解説**: 第5章

第5章で見た、6体の読み取り専用サブエージェントです。`~/.claude/agents/` に、1体ずつ `.md` ファイルとして置きます。frontmatter の `tools:` で使えるツールが絞られている点が要です——Edit も Write も無いので、彼らは修正したくても手が届きません（第2章「権限の非対称」）。`model:` は各エージェントの負荷に合わせた既定値で、環境に応じて変えて構いません。

`~/.claude/agents/ground-truth-check.md`:

```markdown
---
name: ground-truth-check
description: セッション開始直後、またはSTATEファイル・作業指示を受け取った直後に積極的に使う。git log・git statusと実体を確認し、渡された前提との食い違いの有無を返す。読み取り専用。
tools: Bash, Read, Grep, Glob
model: haiku
---
あなたはグラウンドトゥルース確認の専門エージェントです。

手順:
1. `git log --oneline -5` と `git status` を実行
2. 呼び出し元から渡された前提（HEAD・ブランチ・進捗・untrackedファイル等の認識）と実体を突き合わせる
3. 出力は2形式のみ: 「実体確認OK（HEAD <hash> / branch <name> / status要点1行）」または「食い違いあり: <前提> ↔ <実体>」の列挙

禁止事項: ファイルの作成・編集・削除／git add・commit・push・checkout等の変更系コマンド／プロセスの起動・停止。
根拠は必ず実体（ハッシュ・ブランチ名・ファイル名）で示す。推測で埋めない。
```

`~/.claude/agents/explorer.md`:

```markdown
---
name: explorer
description: コードベースの調査・検索に使う。実装箇所の特定、関数の呼び出し元の列挙、特定パターンを含むファイルの探索など、変更を伴わない読み取り全般。ファイルパス・行番号・該当コード片の事実列挙を返す。
tools: Read, Grep, Glob
model: haiku
---
あなたはコードベース探索の専門エージェントです。依頼された調査を行い、事実（ファイルパス・行番号・該当コード片）を簡潔に列挙します。

- 見つからなければ「見つからない」と明確に報告（推測で埋めない）
- 解釈や修正提案は最小限、事実の列挙を優先
- ファイルの作成・編集は一切しない
```

`~/.claude/agents/contrast-debugger.md`:

```markdown
---
name: contrast-debugger
description: 原因不明の不具合・描画異常・想定外の挙動の調査に使う。修正仮説を試す前に必ず先にこれを使う。正常に動く参照との対照実験で、原因の層（データ/モデル側・コード/レンダラ側・環境側）を1つに絞った切り分け結果を返す。修正はしない。
tools: Bash, Read, Grep, Glob
model: sonnet
---
あなたは対照実験専門の調査エージェントです。目的は「直すこと」ではなく「原因の層を1回の調査で確定させること」です。

手順:
1. 症状を1文で確認
2. 「正常に動く参照」を特定する（動作していた過去コミット・公式サンプル・類似の動く実装）。見つからなければ、何を参照にすべきか呼び出し元へ質問して終了
3. 問題側と参照側を同一条件で並べ、差分・ログ・出力を実際に取得して比較（推測禁止）
4. 原因の層を1つに絞り、根拠と共に報告。絞れない場合は「次に対照すべきもの」を1つ提案

禁止事項: 修正の適用(提案は1行まで可)／対照実験なしで仮説修正案を並べること／ファイル変更・git変更系コマンド。
```

`~/.claude/agents/test-runner.md`:

```markdown
---
name: test-runner
description: 実装・修正の直後にテストを実行して結果を確認するときに積極的に使う。指定されたテストスイートを実行し、成否サマリと失敗テストのエラー原文抜粋を返す。修正はしない。
tools: Bash, Read
model: haiku
---
あなたはテスト実行専門のエージェントです。

手順:
1. 指定されたテストコマンドを実行(未指定ならリポジトリの慣習を確認して pytest 等を選ぶ)
2. 結果を「passed/failed/error 件数」で集計
3. 失敗があれば、テスト名と失敗箇所のエラー原文(アサーション・トレースの核心部のみ)を抜粋。長大な出力の全文貼り付けはしない
4. 修正提案は1行まで。修正自体は絶対にしない

禁止事項: ファイル・テストコードの変更／git変更系コマンド。
```

`~/.claude/agents/code-reviewer.md`:

```markdown
---
name: code-reviewer
description: コミット前・人間への検収依頼前のコードレビューに積極的に使う。差分を読み、仕様適合・指示範囲外の変更(スコープ逸脱)・バグ・エッジケース・プロジェクト固有ルール違反を重要度順で返す。読み取り専用。
tools: Read, Grep, Glob, Bash
model: sonnet
---
あなたはコードレビュー専門のエージェントです。新鮮な目で差分だけを見て判断します。

手順:
1. ~/.claude/CLAUDE.md と、リポジトリ直下に CLAUDE.md があればそれを読み、共通規律と固有の禁止事項を把握する
2. レビュー対象の差分を取得する(未コミットなら `git diff`、直近コミットなら `git show`。呼び出し元の指定があればそれに従う)
3. 以下の観点で確認する:
   a. 渡された仕様・指示との適合
   b. スコープ逸脱(指示された「触ってよい範囲」外のファイル変更・巻き込み)
   c. バグ・エッジケース・エラーハンドリングの欠落
   d. CLAUDE.md記載の固有ルール違反(禁止パス・事故防止事項)
   e. ランタイム生成物・大容量ファイル・gitignore対象の混入
4. 出力: CRITICAL / HIGH / MEDIUM / LOW の重要度順に、ファイル:行の参照付きで列挙。各指摘に最小の修正案を1行。問題なければ「レビューOK」と確認済み観点を1行

禁止事項: ファイルの書き換え(提案のみ)／git変更系コマンド。
```

`~/.claude/agents/pre-merge-check.md`:

```markdown
---
name: pre-merge-check
description: コミット直後・マージ前・セッション終了前に積極的に使う。push漏れ・宙に浮いた変更・意図しないファイルの混入を確認した結果を返す。読み取り専用。
tools: Bash, Read
model: haiku
---
あなたはマージ前チェック専門のエージェントです。以下を実行し報告するだけです。

1. `git status` — 未コミット・未追跡ファイル
2. `git branch -vv` — 追跡ブランチとahead/behind(push漏れ検出)
3. `git log --oneline -5` — 直近コミットメッセージの妥当性
4. `git show --stat HEAD` — 意図しないファイルの混入(大量ファイル・runtime生成物・gitignore対象)

問題があれば具体的に指摘(例:「origin/devより2コミット先行・未push」)。無ければ「マージ前チェックOK」とだけ報告。

禁止事項: push・commit・merge・checkout等の変更系コマンド。
```

### commands/ground-truth.md（着手前の実体確認・定型）

**書き換える箇所**: なし（手順・出力形式とも汎用。プロセス確認例も一般名のみ）
**本文の解説**: 第5章

第5章で「確認を測定に置き換える」装置と呼んだコマンドです。`~/.claude/commands/ground-truth.md` に置くと、`/ground-truth` として呼べます。出力を2形式に固定することで、「なんとなく確認した」の置き場所を消します。

```markdown
---
description: 着手前のグラウンドトゥルース確認を定型実行する。git log / status / branch -vv / worktree / 関連プロセスを実体確認し、渡された前提との食い違いを報告する。読み取り専用
argument-hint: (省略可。HEAD・ブランチ等の前提があれば書くと実体と突き合わせる)
---

着手前のグラウンドトゥルース確認を行う。完全に読み取り専用: ファイルの作成・編集・削除、git の変更系コマンド(add/commit/push/checkout等)、プロセスの起動・停止は一切しない。

カレントの作業ディレクトリで以下を順に実行する:

1. `git log --oneline -5`
2. `git status`
3. `git branch -vv` (追跡ブランチと ahead/behind)
4. `git worktree list` (worktree構成のあるリポジトリでは、どのworktreeがどのブランチをロックしているかを確認する)
5. 関連プロセス確認 — このリポジトリのコードを実行中のプロセスの有無を確認する
   (対象リポジトリのパスをCommandLineに含むものだけ報告する。停止はしない。稼働中プロセスは旧コードを保持している点に留意)
6. git リポジトリでない場合は 1〜4 をスキップし、その旨とディレクトリの実在・内容の要点を報告する

前提との突き合わせ: $ARGUMENTS
(引数・STATEファイル・作業指示で HEAD・ブランチ・working tree の状態などの前提が渡されている場合、実体と突き合わせる)

出力は2形式のみ:
- 「実体確認OK (HEAD <hash> / branch <name> / status要点1行 / worktree・プロセスの要点)」
- 「食い違いあり: <前提> ↔ <実体>」の列挙

食い違いがあれば作業を止めて報告する(実体が正)。根拠は必ず実体(ハッシュ・ブランチ名・ファイル名・PID)で示し、推測で埋めない。
```

### commands/handoff.md（実装役へのハンドオフ・定型）

**書き換える箇所**:
- `<稼働中の本体リポジトリ>` … 絶対に触らせたくない、稼働中の本番リポジトリのパス
- `<外部エンジン等の依存先>` … 触れてはいけない外部依存のパス
- `<旧・退役したリポジトリ>` … 退役済みで触れてほしくないパス
- `<spike用ディレクトリ>` … 使い捨て検証を隔離する場所のパス

**本文の解説**: 第2章・第6章

第2章で「拒否権つきのインターフェース」と呼んだ定型です。渡したタスク概要に、実行環境・git規律・禁止パス・完了条件の型を機械的に装着します。末尾の一文——不完全な指示なら着手前に突き返せ——が、指示の品質を送り手の注意力ではなくプロトコルの構造で担保します。禁止パスは環境が割れるためプレースホルダにしてあります。

```markdown
---
description: 三役体制(PO/設計役/実装役)のハンドオフ定型を展開する。引数のタスク概要に、実行環境・git規律・禁止パス・スパイク隔離・完了条件の型・既知ミス注意を装着して作業を開始する
argument-hint: [タスク概要(作業ディレクトリ・触ってよい範囲・完了条件を含めること)]
disable-model-invocation: true
---

以下は三役体制(PO / 設計役 / 実装役)の標準ハンドオフである。あなたは実装役として、下記の共通規律の下で「作業内容」を実行する。

## 実行環境
- 作業ディレクトリ: 作業内容の指定に従う。指定が無ければ着手せず確認を求める
- 着手前に必ずグラウンドトゥルースを取る: `git log --oneline -5` / `git status` / `git branch -vv`(カスタムコマンド /ground-truth があればそれを使う)。渡された指示・STATEと実体が食い違ったら、作業を止めて報告する(実体が正)
- サブエージェント標準フロー(ground-truth-check → explorer / contrast-debugger → 実装 → test-runner → code-reviewer → pre-merge-check)は ~/.claude/CLAUDE.md の規定に従う

## git規律(フックで機械的に強制。ブロックされたら正常動作なので回避せず従う)
- git add は対象ファイルの明示指定のみ。`git add -A` / `git add .` 禁止
- コミットメッセージはファイル経由: 一時ファイルに書いて `git commit -F <file>` → 使用後に削除。heredoc 禁止
- push は指示された作業ブランチへ。main への直接 push・main へのマージは禁止(マージは人間=POの検収後)
- push 後に `git branch -vv` で [origin/...] の追跡状態を確認し、報告に含める(push忘れ防止)

## 触ってはいけない範囲(絶対禁止・全タスク共通)
- <稼働中の本体リポジトリ>(稼働中の本体worktree・プロセス含む) / <外部エンジン等の依存先> / <旧・退役したリポジトリ>
- ~/.claude/ 配下の変更(読み取りは可)
- 触ってよい範囲は作業内容の【書き込み可】指定のみ。指定外は変更しない。範囲外の変更が必要になったら、変更せず理由を報告してから判断を仰ぐ

## スパイク隔離
- 使い捨て検証は <spike用ディレクトリ> 配下で行い、本番リポジトリに混ぜない
- スパイク内の git はローカルのみ・push しない

## プロセス・コスト
- 稼働中プロセスは旧コードを保持する。コード反映は再起動で行う(指示がなければ再起動はPOが行う)
- プロセス停止が指示された場合は Name と CommandLine の両方で厳密一致させる。緩いフィルタ禁止
- API課金が走る処理は、実行前に概算を報告して承認を待つ

## 作業内容
$ARGUMENTS

上記に「作業ディレクトリ」「触ってよい範囲(【書き込み可】)」「完了条件」が含まれていない場合は、着手前に不足項目を列挙して確認を求めること。

## 完了条件の型(作業内容の完了条件に加えて必ず満たす)
- 完了報告に 変更ファイル一覧・コミットハッシュ・push先ブランチ を含める
- push した場合は `git branch -vv` の出力で追跡状態を示す
- main へのマージをしていないこと(マージはPO検収後に人間が行う)
- 「ビルド成功」と「表示・動作が正しい」は別物として報告する。見た目・実機挙動の最終確認はPOの目視である旨を報告に明記する

## 実装役の既知ミス(実例あり・特に注意)
- push忘れ(コミットだけしてpushしない) → push後の `git branch -vv` 確認を省略しない
- heredoc によるコミットメッセージ破損 → 必ず一時ファイル + `-F`
- 記憶で書く → 実装・修正の前に必ず実ファイルを読む。行番号は実体で確認する(ズレの可能性を常に見る)
- 「無い/できない」「済んだ」の自己申告 → 実体(--help・公式ドキュメント・git fetch後のorigin)で裏を取ってから確定する
```

### commands/state-reflect.md（STATE更新・定型）

**書き換える箇所**:
- `<ノートVault>/state/` … STATEのローカル写しを置くノート保管庫のパス（git管理外の場合）

**本文の解説**: 第4章

第4章のSTATE運用を起動する定型です。完成版の全文出力・変更ログ書式の厳守・実体優先という原則を装着します。ローカル写しの保管場所だけ環境依存なのでプレースホルダにしてあります。

```markdown
---
description: STATEファイルを state-update スキルで更新する定型。完成版丸ごと出力・変更ログ書式(- YYYY-MM-DD:)・実体優先・ローカル写し非コミットの原則を装着する
argument-hint: [対象STATEファイルのパス + 反映したい実績・決定事項]
---

state-update スキルを起動して、対象STATEファイルを更新する。

対象と反映内容: $ARGUMENTS
(対象STATEが特定できない場合はリポジトリの CLAUDE.md を確認し、それでも不明ならユーザーに確認する)

スキルの手順に加えて、以下を厳守する:

- 出力は完成版ファイルの全文を丸ごと書き出す。要約・差分では返さない
- STATEのローカル写しが <ノートVault>/state/ 等にある場合、そこが git 管理外なら commit しない(ファイル書き出しのみ)。リポジトリ内のSTATEを更新した場合のみ、git規律(targeted add・-F)に従ってコミットする
- 変更ログへの追記は行頭 `- YYYY-MM-DD:` 書式を厳守する(開発日誌の自動転記がこの書式に依存)
- 情報の優先順位は 実体(git log・実プロセス・ファイル) > STATE > 渡された指示。食い違いを見つけたら実体を正とし、食い違いがあった事実もSTATEに記録する
- 「マージ済み」「PR済み」等の状態は、`git fetch` 後の origin 実体(`git branch -r --contains <commit>` 等)で確認してから書く。未マージなら「feat/xxx 完了・dev未マージ・PO検収待ち」のように正確な状態で書く
- 確定事項・決着済みに入れる決定には理由を添える(蒸し返し防止)。「次のアクション」は次のセッションがそこから着手できる具体性で書く
```

---

## 7.3 第3層：資産化（STATE運用が前提）

この層は、第4章のSTATE運用がすでに回っていることを前提にします。STATEを書式どおりに書き続けているなら、その蓄積を機械が資産へ変えられます。

**この層には、一つ警告があります。** これから示す `dev_diary.py` は、STATEの変更ログの書式——行頭が `- YYYY-MM-DD:` で始まる一行——に依存します。第4章で「宣言された結合」と呼んだものが、ここに実物として現れます。この書式を守らないと、その日の記述は転記されないだけでなく、その日を「何も起きなかった日」に見せることすらあります。だからこの層を入れるなら、7.2 の `state-reflect` を通してSTATEを更新する運用を、先に固めておいてください。書式は美学ではなく、これらのスクリプトとの契約です。

### skills/state-update/SKILL.md（STATE更新スキル本体）

**書き換える箇所**:
- `<PROJECT>_STATE.md` … あなたのプロジェクトのSTATEファイル名（例を自分のものに）

**本文の解説**: 第4章

`state-reflect` コマンドが呼び出すスキルの本体です。`~/.claude/skills/state-update/SKILL.md` に置きます。差分ではなく全文書き直しを指示する手順4が核心です（第4章）。書き手が一度コストを払い、読み手の検証コストをゼロにする前払いです。

```markdown
---
name: state-update
description: STATEファイル(<PROJECT>_STATE.md)を更新するときに使う。セッションで意味のある進捗があった直後、ユーザーが「STATEを更新して」と依頼したとき、またはセッションを終える前に積極的に使う。
---

STATEファイルを更新する手続きです。以下の順で行い、最後に完成版ファイルを書き出します。

1. 対象のSTATEファイルを特定する(リポジトリのCLAUDE.mdに記載があればそれに従う。無ければユーザーに確認する)
2. 現在のSTATEファイルを全文読む
3. 変更点を集める:
   - `git log --oneline` を、STATEの「最終更新」日以降に絞って確認する
   - `git status` で未コミットの変更も確認する
   - このセッション中にユーザーと合意した決定事項・却下した案とその理由も含める
4. STATEファイルを完成版で書き直す。既存の章立て(プロジェクト定義／確定事項／現在地／宿題トラッキング／次のアクション／決着済み／変更ログ)は維持し、差分ではなく全文を作り直す
5. 冒頭の「最終更新」日付を今日の日付に更新する
6. 末尾の変更ログに、行頭が `- YYYY-MM-DD:` で始まる1行を追記する(この書式は開発日誌の自動転記が依存しているため厳守)
7. 確定事項・決着済みに入れた決定には、理由も一緒に書く(蒸し返し防止)
8. 「次のアクション」は、次にこのSTATEを開いた人がそこから着手できる具体性で書く

出力は完成版ファイルの全文。要約や差分では返さない。
```

### dev_diary.py（git履歴とSTATE変更ログから日誌を生成）

**書き換える箇所**（すべて先頭の設定ブロックに集約してあります）:
- `REPOS` … 日誌に集約したいリポジトリ（名前 → パス）
- `STATE_FILES` … 変更ログを拾うSTATEファイル（見出し → パス）
- `OUT_DIR` … 生成した日誌を書き出す先

**本文の解説**: 第4章

第4章で「散文に機械が依存している」実例として登場したスクリプトです。ネットワーク呼び出しもLLM呼び出しもなく、ローカルの git log とSTATEの変更ログを読んで、日付ごとのMarkdownノートを生成・更新するだけです。環境依存の値は先頭の設定ブロックに集約し、それ以外は触らずに動くようにしてあります。

`collect_state_section` の中の正規表現 `^- {date}\b` に注目してください。ここが、第4章で説いた「事実上のAPIスキーマ」の実体です。STATEの変更ログがこの書式を外すと、この行は拾われません。

```python
"""Dev diary generator: turns git history into daily notes.

No network calls, no LLM. Reads git log locally and writes markdown files.
"""
import argparse
import re
import subprocess
from datetime import datetime, timedelta
from pathlib import Path

# ==== 書き換える箇所（ここだけ自分の環境に合わせる） =======================
# 日誌に集約したい git リポジトリ（表示名 -> リポジトリの絶対パス）
REPOS = {
    "<repo1>": Path(r"<repo1 への絶対パス>"),
    "<repo2>": Path(r"<repo2 への絶対パス>"),
}
# 変更ログを拾う STATE ファイル（日誌上の見出し -> STATE の絶対パス）
STATE_FILES = {
    "<PROJECT>": Path(r"<PROJECT_STATE.md への絶対パス>"),
}
# 生成した日誌 md を書き出すディレクトリ
OUT_DIR = Path(r"<日誌の出力ディレクトリ>")
# ==== 書き換えるのはここまで ===============================================

DATE_FMT = "%Y-%m-%d"

# ---- git collection --------------------------------------------------------

def collect_git_for_repo(repo_path: Path, date: datetime):
    if not repo_path.exists():
        print(f"[warn] repo path not found, skipping: {repo_path}")
        return None
    since = date.strftime(f"{DATE_FMT} 00:00")
    until = (date + timedelta(days=1)).strftime(f"{DATE_FMT} 00:00")
    try:
        result = subprocess.run(
            [
                "git", "-C", str(repo_path), "log", "--all",
                f"--since={since}", f"--until={until}",
                "--date=format-local:%H:%M",
                "--pretty=format:%h|%ad|%s",
            ],
            capture_output=True, text=True, encoding="utf-8", errors="replace",
        )
    except FileNotFoundError:
        print("[warn] git executable not found")
        return None
    if result.returncode != 0:
        print(f"[warn] git log failed for {repo_path}: {result.returncode}")
        return None
    lines = [l for l in result.stdout.splitlines() if l.strip()]
    commits = []
    for line in lines:
        parts = line.split("|", 2)
        if len(parts) != 3:
            continue
        commits.append(tuple(parts))  # (hash, hhmm, subject)
    commits.reverse()  # newest-first -> chronological
    return commits


def collect_all_git(date: datetime):
    out = {}
    for name, path in REPOS.items():
        commits = collect_git_for_repo(path, date)
        if commits:
            out[name] = commits
    return out


def format_git_section(git_by_repo: dict) -> str:
    if not git_by_repo:
        return "(コミットなし)"
    blocks = []
    for name in REPOS:  # keep configured order
        commits = git_by_repo.get(name)
        if not commits:
            continue
        lines = [f"### {name}"]
        for h, hhmm, subject in commits:
            lines.append(f"- {hhmm} `{h}` {subject}")
        blocks.append("\n".join(lines))
    return "\n\n".join(blocks) if blocks else "(コミットなし)"


# ---- state collection ------------------------------------------------------

def collect_state_section(date: datetime) -> str:
    date_str = date.strftime(DATE_FMT)
    if not STATE_FILES:
        return "(該当なし)"
    lines_out = []
    # ↓ この行頭パターンが STATE 変更ログ書式との「契約」。外すと拾われない。
    pattern = re.compile(rf"^- {re.escape(date_str)}\b")
    for name, path in STATE_FILES.items():
        if not path.exists():
            print(f"[warn] state file not found, skipping: {path}")
            continue
        text = path.read_text(encoding="utf-8", errors="replace")
        matched = [l for l in text.splitlines() if pattern.match(l.strip())]
        if matched:
            lines_out.append(f"### {name}")
            lines_out.extend(matched)
    return "\n".join(lines_out) if lines_out else "(該当なし)"


# ---- file template / marker handling ---------------------------------------

MARKERS = ["git", "state", "chat"]


def marker_block(marker: str, body: str) -> str:
    return f"<!-- {marker}:start -->\n{body}\n<!-- {marker}:end -->"


def build_skeleton(date: datetime, git_body: str, state_body: str) -> str:
    date_str = date.strftime(DATE_FMT)
    parts = [
        f"# {date_str} 開発日誌",
        "",
        marker_block("git", f"## やったこと（git）\n\n{git_body}"),
        "",
        "## 背景・判断",
        "",
        marker_block("state", state_body),
        "",
        marker_block("chat", ""),
        "",
        "## メモ",
        "",
    ]
    return "\n".join(parts) + "\n"


def replace_marker(content: str, marker: str, new_body: str) -> str:
    pattern = re.compile(
        rf"(<!-- {marker}:start -->\n).*?(\n<!-- {marker}:end -->)", re.DOTALL
    )
    if not pattern.search(content):
        raise ValueError(f"marker {marker} not found in existing file")
    return pattern.sub(lambda m: m.group(1) + new_body + m.group(2), content, count=1)


def daily_file_path(date: datetime) -> Path:
    return OUT_DIR / f"{date.strftime(DATE_FMT)}.md"


def upsert_daily(date: datetime, update_chat_body: str = None) -> Path:
    """Refresh git+state markers; optionally set chat marker. Creates file if missing."""
    OUT_DIR.mkdir(parents=True, exist_ok=True)
    path = daily_file_path(date)

    git_by_repo = collect_all_git(date)
    git_section = format_git_section(git_by_repo)
    git_body = f"## やったこと（git）\n\n{git_section}"
    state_body = collect_state_section(date)

    if path.exists():
        content = path.read_text(encoding="utf-8", errors="replace")
        content = replace_marker(content, "git", git_body)
        content = replace_marker(content, "state", state_body)
        if update_chat_body is not None:
            content = replace_marker(content, "chat", update_chat_body)
    else:
        content = build_skeleton(date, git_section, state_body)
        if update_chat_body is not None:
            content = replace_marker(content, "chat", update_chat_body)

    path.write_text(content, encoding="utf-8")
    return path


def has_activity(date: datetime) -> bool:
    for name, path in REPOS.items():
        if collect_git_for_repo(path, date):
            return True
    if STATE_FILES:
        date_str = date.strftime(DATE_FMT)
        pattern = re.compile(rf"^- {re.escape(date_str)}\b")
        for name, path in STATE_FILES.items():
            if not path.exists():
                continue
            text = path.read_text(encoding="utf-8", errors="replace")
            if any(pattern.match(l.strip()) for l in text.splitlines()):
                return True
    return False


# ---- backfill --------------------------------------------------------------

def earliest_commit_date() -> datetime:
    earliest = None
    for name, path in REPOS.items():
        if not path.exists():
            continue
        result = subprocess.run(
            [
                "git", "-C", str(path), "log", "--all", "--reverse",
                "--date=format-local:%Y-%m-%d",
                "--pretty=format:%ad",
            ],
            capture_output=True, text=True, encoding="utf-8", errors="replace",
        )
        lines = [l for l in result.stdout.splitlines() if l.strip()]
        if lines:
            d = datetime.strptime(lines[0], DATE_FMT)
            if earliest is None or d < earliest:
                earliest = d
    return earliest


def run_backfill():
    earliest = earliest_commit_date()
    if earliest is None:
        print("[warn] no commits found in any repo, nothing to backfill")
        return
    today = datetime.now().replace(hour=0, minute=0, second=0, microsecond=0)
    print(f"backfill from {earliest.strftime(DATE_FMT)} to {today.strftime(DATE_FMT)}")
    count = 0
    d = earliest
    while d <= today:
        if has_activity(d):
            upsert_daily(d)
            count += 1
        d += timedelta(days=1)
    print(f"backfill done, generated/updated {count} day(s)")


# ---- merge-chat ------------------------------------------------------------

DATE_HEADING_RE = re.compile(r"^## (\d{4}-\d{2}-\d{2})\s*$", re.MULTILINE)


def run_merge_chat(chat_md_path: Path):
    text = chat_md_path.read_text(encoding="utf-8", errors="replace")
    matches = list(DATE_HEADING_RE.finditer(text))
    if not matches:
        print("[warn] no '## YYYY-MM-DD' headings found in chat file")
        return
    count = 0
    for i, m in enumerate(matches):
        date_str = m.group(1)
        start = m.end()
        end = matches[i + 1].start() if i + 1 < len(matches) else len(text)
        body = text[start:end].strip("\n")
        try:
            date = datetime.strptime(date_str, DATE_FMT)
        except ValueError:
            print(f"[warn] bad date heading skipped: {date_str}")
            continue
        upsert_daily(date, update_chat_body=body)
        count += 1
    print(f"merge-chat done, updated {count} day(s)")


# ---- CLI -------------------------------------------------------------------

def main():
    parser = argparse.ArgumentParser(description="Generate dev diary from git history")
    parser.add_argument("--date", help="YYYY-MM-DD")
    parser.add_argument("--backfill", action="store_true")
    parser.add_argument("--merge-chat", help="path to chat markdown file")
    args = parser.parse_args()

    if args.merge_chat:
        run_merge_chat(Path(args.merge_chat))
        return
    if args.backfill:
        run_backfill()
        return
    if args.date:
        date = datetime.strptime(args.date, DATE_FMT)
    else:
        date = datetime.now().replace(hour=0, minute=0, second=0, microsecond=0)

    upsert_daily(date)
    print(f"wrote diary for {date.strftime(DATE_FMT)}")


if __name__ == "__main__":
    main()
```

（実運用版には、特定リポジトリの停滞を検知して通知する付加機能もありますが、通知経路が環境に強く依存し、本文の主題からも外れるため、付録では移植しやすい中核だけを載せました。毎日決まった時刻に走らせたい場合は、OSのタスクスケジューラや cron から無引数で呼ぶだけで、その日の分が生成されます。過去にさかのぼって一括生成したいときは `--backfill` を使います。）

### scan.py（リポジトリ群を横断して環境マップを一枚出す）

**書き換える箇所**（すべて先頭の設定ブロックに集約してあります）:
- `CLAUDE_ROOT` … 横断スキャンするリポジトリ群のルートディレクトリ
- `VAULT_STATE_DIR` … STATEを集約している保管庫のディレクトリ（無ければ空でも可）
- `CLAUDE_HOME` … Claude Code の設定ディレクトリ（`~/.claude` の実体）

**本文の解説**: 第4章

環境全体を一枚のデータに落とすスキャナです。ルート直下で git 管理下のリポジトリを動的に列挙し（ハードコードしません）、各リポジトリの現在ブランチ・直近コミット・未push状態・STATEの最終更新日を集めます。AI呼び出しは無く、git 照会とファイル読み取りとJSON書き出しだけなので、API課金はゼロです。

ここでも、STATEの変更ログから最新日付を拾う正規表現 `^- (\d{4}-\d{2}-\d{2})` が中核にあります。第4章の「書式は契約」が、日誌生成だけでなく、環境マップの鮮度判定でも効いているのが分かります。

```python
"""env-map scan: リポジトリ群を横断して env_map.json を1本吐く（読み取り専用）。

git 照会 / ファイル読取 / JSON 書き出しのみ。AI 呼び出しなし = API 課金ゼロ。
書き込み先は本スクリプトと同じディレクトリの env_map.json のみ。

前提:
- 対象 = ルート直下で .git を持つリポジトリ全て（動的列挙・ハードコードしない）。
- STATE は repo-root 直下(*_STATE.md) と 保管庫集約 の両方を見て新しい方を採用。
  両方存在して日付が食い違えば divergence=true を立てて記録（黙って握り潰さない）。
- STATE 変更ログ書式: `## 変更ログ` 見出し下に `- YYYY-MM-DD:`。
"""
import json
import re
import subprocess
from datetime import datetime
from pathlib import Path

# ==== 書き換える箇所（ここだけ自分の環境に合わせる） =======================
CLAUDE_ROOT = Path(r"<横断スキャンするリポジトリ群のルート>")
VAULT_STATE_DIR = Path(r"<STATEを集約している保管庫のディレクトリ>")
CLAUDE_HOME = Path(r"<Claude Code の設定ディレクトリ（~/.claude の実体）>")
# ==== 書き換えるのはここまで ===============================================

OUT_PATH = Path(__file__).resolve().parent / "env_map.json"
OUT_JS = Path(__file__).resolve().parent / "env_map.js"  # ビューアが読む自動データ

DATE_RE = re.compile(r"^- (\d{4}-\d{2}-\d{2})\b")
CHANGELOG_HEADING = "## 変更ログ"
GIT_TIMEOUT = 20  # 秒。ハングした repo で全体を止めないための保険。


# ---- helpers ---------------------------------------------------------------

def read_text_best(path: Path) -> str:
    """文字コードの取り違えで文字化けした経緯があるため多段フォールバック。"""
    for enc in ("utf-8", "cp932"):
        try:
            return path.read_text(encoding=enc)
        except (UnicodeDecodeError, UnicodeError):
            continue
    return path.read_text(encoding="utf-8", errors="replace")


def git(repo_path: Path, args):
    """読み取り専用 git 照会。失敗時は (None, error_str) を返しフェイルオープン。"""
    try:
        result = subprocess.run(
            ["git", "-C", str(repo_path), *args],
            capture_output=True, text=True, encoding="utf-8", errors="replace",
            timeout=GIT_TIMEOUT,
        )
    except FileNotFoundError:
        return None, "git executable not found"
    except subprocess.TimeoutExpired:
        return None, f"git timeout ({GIT_TIMEOUT}s)"
    if result.returncode != 0:
        return None, (result.stderr or "").strip() or f"git exit {result.returncode}"
    return result.stdout, None


# ---- repository discovery --------------------------------------------------

def discover_repos():
    """ルート直下で .git（dir でも file でも）を持つディレクトリを列挙。"""
    repos = []
    for child in sorted(CLAUDE_ROOT.iterdir()):
        if not child.is_dir():
            continue
        if (child / ".git").exists():
            repos.append(child)
    return repos


# ---- STATE resolution ------------------------------------------------------

def changelog_latest_date(text: str):
    """変更ログ節の `- YYYY-MM-DD` から最新(max)日付を返す。無ければ全文から探す。"""
    lines = text.splitlines()
    scan_from = 0
    for i, line in enumerate(lines):
        if line.strip() == CHANGELOG_HEADING:
            scan_from = i + 1
            break
    for start in (scan_from, 0):
        dates = []
        for line in lines[start:]:
            m = DATE_RE.match(line.strip())
            if m:
                dates.append(m.group(1))
        if dates:
            return max(dates)
    return None


def state_candidate(path: Path):
    text = read_text_best(path)
    last_date = changelog_latest_date(text)
    mtime = datetime.fromtimestamp(path.stat().st_mtime)
    return {
        "path": str(path),
        "last_date": last_date,                 # 変更ログ由来（無ければ None）
        "mtime_date": mtime.strftime("%Y-%m-%d"),
    }


def effective_date(c):
    return c["last_date"] or c["mtime_date"]


def resolve_state(repo_name: str, repo_path: Path):
    candidates = {}
    roots = sorted(repo_path.glob("*_STATE.md"))
    if roots:
        candidates["repo-root"] = state_candidate(roots[0])
    vault_path = VAULT_STATE_DIR / f"{repo_name.upper()}_STATE.md"
    if vault_path.exists():
        candidates["vault"] = state_candidate(vault_path)

    if not candidates:
        return {"source": "none", "path": None, "last_date": None, "divergence": False}

    source_name = max(candidates, key=lambda k: effective_date(candidates[k]))
    chosen = candidates[source_name]
    out = {
        "source": source_name,
        "path": chosen["path"],
        "last_date": effective_date(chosen),
        "divergence": False,
    }
    if len(candidates) == 2:
        d_root = effective_date(candidates["repo-root"])
        d_vault = effective_date(candidates["vault"])
        if d_root != d_vault:
            out["divergence"] = True
            out["divergence_detail"] = {
                "repo_root": {"path": candidates["repo-root"]["path"], "date": d_root},
                "vault": {"path": candidates["vault"]["path"], "date": d_vault},
            }
    return out


# ---- per-repo collection ---------------------------------------------------

def parse_unpushed(branch_vv_output: str):
    """現在ブランチ行(先頭 *)に [origin/...] upstream が無ければ unpushed=true。"""
    if branch_vv_output is None:
        return None, None
    current_line = None
    for line in branch_vv_output.splitlines():
        if line.startswith("*"):
            current_line = line
            break
    if current_line is None:
        return None, None
    has_origin_upstream = re.search(r"\[origin/[^\]]+\]", current_line) is not None
    return (not has_origin_upstream), current_line.rstrip()


def collect_repo(repo_path: Path):
    name = repo_path.name
    info = {"name": name, "path": str(repo_path), "errors": []}

    branch_out, err = git(repo_path, ["rev-parse", "--abbrev-ref", "HEAD"])
    info["current_branch"] = branch_out.strip() if branch_out else None
    if err:
        info["errors"].append(f"current_branch: {err}")

    latest_out, err = git(repo_path, ["log", "--oneline", "-1"])
    info["latest_commit"] = latest_out.strip() if latest_out else None
    if err:
        info["errors"].append(f"latest_commit: {err}")

    vv_out, err = git(repo_path, ["branch", "-vv"])
    if err:
        info["errors"].append(f"unpushed: {err}")
    unpushed, current_line = parse_unpushed(vv_out)
    info["unpushed"] = unpushed

    info["state"] = resolve_state(name, repo_path)

    if not info["errors"]:
        del info["errors"]
    return info


# ---- infra layer -----------------------------------------------------------

def collect_hooks():
    """設定ディレクトリの settings.json の hooks を種別+対象ツール+コマンド粒度で列挙。"""
    settings_path = CLAUDE_HOME / "settings.json"
    if not settings_path.exists():
        return {"source": str(settings_path), "error": "settings.json not found", "entries": []}
    try:
        data = json.loads(read_text_best(settings_path))
    except json.JSONDecodeError as e:
        return {"source": str(settings_path), "error": f"json parse failed: {e}", "entries": []}

    hooks = data.get("hooks", {})
    entries = []
    if isinstance(hooks, dict):
        for event, matchers in hooks.items():
            if not isinstance(matchers, list):
                continue
            for m in matchers:
                if not isinstance(m, dict):
                    continue
                cmds = []
                for h in m.get("hooks", []) or []:
                    if isinstance(h, dict):
                        cmds.append({"type": h.get("type"), "command": h.get("command")})
                entries.append({"event": event, "matcher": m.get("matcher"), "hooks": cmds})
    return {"source": str(settings_path), "entries": entries}


def collect_agents():
    """設定ディレクトリの agents/ の *.md をエージェント名として列挙。"""
    agents_dir = CLAUDE_HOME / "agents"
    if not agents_dir.exists():
        return {"source": str(agents_dir), "error": "agents dir not found", "names": []}
    names = sorted(p.stem for p in agents_dir.glob("*.md"))
    return {"source": str(agents_dir), "names": names}


# ---- main ------------------------------------------------------------------

def build_env_map():
    repos = discover_repos()
    return {
        "scanned_at": datetime.now().isoformat(timespec="seconds"),
        "claude_root": str(CLAUDE_ROOT),
        "repositories": [collect_repo(r) for r in repos],
        "infra": {"hooks": collect_hooks(), "agents": collect_agents()},
        "external": [],   # 外部連携層: scan は自動収集しない。手動で埋める領域。
        "relations": [],  # 層またぎの関係: 後で手書きする領域。
    }


def main():
    env_map = build_env_map()
    OUT_PATH.write_text(json.dumps(env_map, ensure_ascii=False, indent=2), encoding="utf-8")
    # ビューア用の自動データ。fetch(file://) 非依存にするため変数に載せて出す。
    OUT_JS.write_text(
        "window.ENV_MAP = " + json.dumps(env_map, ensure_ascii=False) + ";\n",
        encoding="utf-8",
    )
    repos = env_map["repositories"]
    unpushed = [r["name"] for r in repos if r.get("unpushed") is True]
    divergent = [r["name"] for r in repos if r["state"].get("divergence")]
    print(f"wrote {OUT_PATH}")
    print(f"repositories: {len(repos)}")
    print(f"unpushed ({len(unpushed)}): {', '.join(unpushed) if unpushed else '(none)'}")
    print(f"state divergence ({len(divergent)}): {', '.join(divergent) if divergent else '(none)'}")


if __name__ == "__main__":
    main()
```

自動収集する層（リポジトリの状態・フック・エージェント）と、手で埋める層（外部連携・層またぎの関係）を、`external` / `relations` として分けてあるのが設計の要です。機械が確実に取れるものだけを自動化し、判断の要る関係づけは人間に残す——第6章で見た、自動と手動の分離の思想が、ここにも通っています。

### README（復元手順）

**書き換える箇所**:
- `<設定ディレクトリ>` … Claude Code の設定ディレクトリ（`~/.claude` の実体）
- `<日誌スクリプトの置き場>` … `dev_diary.py` を配置する場所
- `<定期実行タスク名>` … スケジューラに登録するタスクの名前

この付録の一式は、1台のマシンにしか存在しない設定です。つまり単一障害点です。第1章から繰り返してきた「実体を確認する」という思想は、設定そのもののバックアップにも向けられるべきでしょう。ここだけは、外部から貼り付ける素材ではなく、この付録の中で完結する運用手順として書いておきます。

`~/.claude/` 配下の設定と `dev_diary.py` / `scan.py` を、別のリポジトリへ都度コピーして退避します（自動同期ではなく、意図的な手動バックアップです）。新しいマシンで復元する手順は、次のとおりです。

1. `CLAUDE.md` を `<設定ディレクトリ>/CLAUDE.md` へコピーする
2. `agents/*.md`（6体）を `<設定ディレクトリ>/agents/` へコピーする
3. `skills/state-update/` を `<設定ディレクトリ>/skills/state-update/` へコピーする
4. `commands/*.md`（`ground-truth` / `handoff` / `state-reflect`）を `<設定ディレクトリ>/commands/` へコピーする
5. `settings.json` の `hooks` キー（PreToolUse 4本 + SessionStart）を `<設定ディレクトリ>/settings.json` にマージする。既存の他のキーは残し、`hooks` だけを移植する
6. `dev_diary.py` / `scan.py` を `<日誌スクリプトの置き場>` へコピーし、`dev_diary.py` の先頭の設定ブロックを新しい環境のパスへ書き換える。毎日決まった時刻に走らせるなら、OSのスケジューラに `<定期実行タスク名>` として登録する

復元のたびに、手順6の「先頭の設定ブロックを書き換える」が必要になります。書き換え箇所を先頭に集約し、`<...>` で宣言してあるのは、まさにこの移植のためです。隠された設定は移植のたびに探し回る羽目になりますが、宣言された設定は一箇所を見れば済みます。

---

## 7.4 導入しなかったもの（と、その理由）

付録に何を載せるかと同じくらい、何を載せないかも設計判断です。手元の設定には、この付録に含めなかったものがいくつかあります。理由を添えて、短く記しておきます。

**settings.json の statusLine**。ステータス表示をカスタマイズする設定ですが、手元のものは個人環境の固有パスを指しており、しかも現在はどこからも参照されていないデッドコードでした。動かない見本を付録に載せても、読者を混乱させるだけです。第1章で見たとおり、参照されていないスクリプトは、あると思って頼ると裏切ります。だから外しました。

**settings.local.json（許可リスト）**。ツールの許可リストは、どのコマンドを無確認で通してよいかの設定ですが、これは読者ごとに、作業内容ごとに、まったく異なります。誰かの許可リストをそのまま持ち込むことには意味がなく、むしろ自分の環境に合わない許可を無自覚に引き受けるリスクのほうが大きい。汎用の型として配れないものは、配らないのが誠実です。

**環境マップの出力データそのもの**。`scan.py` は載せましたが、それが吐き出す `env_map.json` のような実データは載せていません。仕組みは汎用で紹介の価値がありますが、出力は特定環境のスキャン結果——リポジトリ名やコミット情報——そのものです。仕組みと、仕組みが生んだデータは、公開可否がまったく違います。前者は環境が割れず、後者は環境が丸ごと割れます。

そして最後に、より大きな判断があります。**この付録には、本文で参照した内部の設計文書や棚卸し記録を、一つも含めていません**。第3章や第5章で引用した棚卸し文書、第6章の引き継ぎ三点セット——それらの原文は、ここにありません。必要な知見は、すでに各章の本文へ、環境が割れない一般形に清書して織り込んであるからです。原文を付録に添付する必要はありません。むしろ、原文には特定環境の固有名が詰まっており、そのまま出せば、これまで払ってきた抽象化の努力を最後の付録で台無しにすることになります。

知見は本文へ、設定は付録へ。データと原文は、どちらにも出さない。この切り分けそのものが、公開という行為に対する一つの規律です。

---

## 結び——設定は、思想の結晶である

第7章で並べたのは、ただの設定ファイルの束ではありません。

フックの正規表現一本には、静かに課金が走り続けた2週間の記憶が畳み込まれています（第3章）。出力を2形式に固定したコマンドには、「見ました、大丈夫そうです」という無為への抵抗が刻まれています（第5章）。全文書き直しを強いるスキルには、新旧の真実が地層をなすことへの警戒が込められています（第4章）。読み取り専用に絞られたエージェント定義には、直せる者に調べさせない、という利害の分離が実装されています（第2章・第5章）。どのファイルも、一度払った授業料の領収書であり、同時に、同じ失敗を二度させないための配線です。

そして、この付録を貫く一つの流儀——書き換える場所を隠さず宣言する——は、本書全体の主題の、最後の反復でした。隠された結合は事故を生み、宣言された結合は運用を支える。盲点は消せないが、記録した盲点は無害化できる。見えないものを補うのは、より賢いモデルではなく、見える場所に置かれた手続きである。第1章のあの結論が、コードと設定という最も具体的な姿をとって、ここに置かれています。

だから、この付録をコピーして自分の環境へ持ち込むとき、あなたが移しているのは設定ではありません。設定という形をとった、いくつもの失敗と、そこから引き出された判断です。`<...>` を自分の環境に書き換えたその瞬間から、これらの配線はあなたのものになります。

体制は、一人でも組めます。必要なのは、高価なツールでも大人数のチームでもなく、部品の組み方と、それを回すための役割の線引きだけでした。その部品の実物を、いま、あなたは手にしています。あとは、最初のフック一本を置くことから始めてください。
