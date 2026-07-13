---
title: "自作MCPサーバーを書いて公開するまで ― Claude × GPT 相互レビューを題材に"
emoji: "🤝"
type: "tech"
topics: ["mcp", "claude", "claudecode", "python", "anthropic"]
published: false
---

## はじめに

近年Anthropic社が公開したModel Context Protocol（MCP）は、LLMから外部のツールやデータソースを安全に利用するための標準プロトコルです（公式: [modelcontextprotocol.io](https://modelcontextprotocol.io)）。2024年11月の公開以降、OpenAI や Google 等も採用する業界標準プロトコルとして定着しつつあります。

本記事のゴールは、自作のMCPサーバーをゼロから書き、Claude Code / Claude Desktopから呼び出し、最終的にuvx経由で誰でも一行起動できる形で配布するまでを一通り体験することです。完成品は筆者がGitHubで公開している [mutual-review-mcp](https://github.com/miharu8686/mutual-review-mcp) を題材にします。

想定読者はPythonの基礎があり、Claude Codeを日常的に触っている方です。

## MCPサーバーの最小構成

このセクションのゴールは、依存ライブラリ `mcp` を使った最短のMCPサーバー実装パターンを掴むことです。

### mcp パッケージが提供するもの

Python用のMCP SDKは `mcp` というパッケージ名で配布されています。本記事の文脈では次の3つを覚えれば十分です。

- `Server`: ツールを登録する中心となるサーバーインスタンス
- `@server.list_tools()` / `@server.call_tool()`: ツール宣言と呼び出しハンドラを定義するデコレータ
- `stdio_server`: 標準入出力でJSON-RPCをやり取りするトランスポート層

stdio版のMCPサーバーは「標準入力からJSON-RPCを受け取り、標準出力で返す」だけのシンプルなプロセスです。HTTPサーバーを立てる必要はなく、Claude Desktopなどのホストがプロセスを起動して直接パイプで会話します。

### stdio サーバーの3ステップ

骨格は3ステップに分解できます。`Server` インスタンスを作り、`@server.list_tools()` と `@server.call_tool()` のデコレータでツールの宣言と実行を登録し、最後に `stdio_server` を `asyncio` で起動するだけです。最小骨格は次のようになります（ツール登録部は次節で具体化します）。

```python
from mcp.server import Server
from mcp.server.stdio import stdio_server
import asyncio

server = Server("my-mcp")

async def main():
    async with stdio_server() as (r, w):
        await server.run(r, w, server.create_initialization_options())
```

### Hello World: インストール済みPythonパッケージ一覧を返すMCP

「LLMが知らないローカル情報をMCPで渡す」というMCPの本質をそのまま体現する例です。標準ライブラリの `importlib.metadata` だけで完結し、外部依存はゼロです。

```python
"""list_packages_mcp.py - インストール済みPythonパッケージ一覧を返すMCP"""
import asyncio
import importlib.metadata
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import TextContent, Tool

server: Server = Server("list-packages")


@server.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="list_installed_packages",
            description="Return name and version of every Python package installed in the runtime.",
            inputSchema={"type": "object", "properties": {}},
        ),
    ]


@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    pkgs = sorted(
        f"{d.metadata['Name']}=={d.version}"
        for d in importlib.metadata.distributions()
    )
    return [TextContent(type="text", text="\n".join(pkgs))]


async def main():
    async with stdio_server() as (r, w):
        await server.run(r, w, server.create_initialization_options())


if __name__ == "__main__":
    asyncio.run(main())
```

これだけで「いまPCにどのPythonパッケージが入っているか教えて」とClaudeに頼んだ際、モデルの学習データではなく、ホストPCの実態が返ってくるようになります。LLMが本来知り得ないローカル情報をMCPでブリッジする、というのが最も基本的な利用パターンです。

### Claude Code に1コマンドで登録

書いたMCPは次の1行で利用できます。

```bash
claude mcp add list-packages -- python /path/to/list_packages_mcp.py
```

これで `/mcp` コマンドにサーバーが現れ、必要に応じてClaudeが自動で `list_installed_packages` を呼び出すようになります。

## mutual-review-mcp で見る実装パターン

このセクションのゴールは、複数LLM・複数ツールを持つ少し現実的なMCPサーバーで意識した設計判断を、実コードから抜粋して見ていくことです。

### 複数LLMの結果を束ねる設計

mutual-review-mcp の中核は、Claude と GPT-4o に独立にレビューを書かせ、その結果を Claude に統合させる三段構成です。

```python
def review_code(code: str, language: str | None = None,
                filename: str | None = None,
                context: str | None = None,
                synthesize_result: bool = True) -> dict:
    if language is None:
        language = guess_language(filename)
    claude_r = review_with_claude(code, filename, context, language)
    gpt_r    = review_with_gpt(code, filename, context, language)
    result: dict = {
        "filename": filename, "language": language,
        "claude": claude_r, "gpt": gpt_r,
    }
    if synthesize_result:
        result["synthesis"] = synthesize(code, claude_r, gpt_r, filename, language)
    return result
```

レビューを2モデルから取って統合する理由は、片方が見落とした観点をもう片方が拾うことが多く、最終的な精度が単一モデルより安定するからです。統合役にClaudeを使っているのは、長文の要約と優先順位付けが安定して上手いという実体験ベースの選択で、必要なら入れ替え可能な構造にしてあります。

### ツールのパラメータ設計とJSON Schema

ツールの良し悪しは、結局のところ「LLMがそのツールを正しく呼べる入力スキーマを書けるか」にかかっています。`review_file` の定義を例に見てみます。

```python
Tool(
    name="review_file",
    description=(
        "Mutual code review on a file. Claude and GPT-4o each review independently, "
        "then Claude synthesizes the findings."
    ),
    inputSchema={
        "type": "object",
        "properties": {
            "path": {"type": "string", "description": "Absolute path to the file"},
            "language": {
                "type": "string",
                "description": "Optional language hint (e.g. python, typescript). Auto-detected from extension if omitted.",
            },
            "context": {"type": "string", "description": "Optional context about the code"},
            "synthesize": {
                "type": "boolean",
                "description": "Generate synthesis report (default true)",
                "default": True,
            },
        },
        "required": ["path"],
    },
),
```

3つのポイントがあります。

- `description` は「いつこのツールを呼ぶべきか」をLLM向けに書きます（人間向けのドキュメントとは粒度が違います）
- `required` は最小限に絞ります。LLMが「埋められる引数だけ埋めて呼ぶ」運用を素直にできる設計が望ましいです
- `default` を明示すると、LLMが省略すべき場面と明示すべき場面を判断しやすくなります

### APIキー管理: 環境変数 → 設定ファイルのフォールバック

MCPサーバーはホスト（Claude Desktop など）から起動されるため、ホストが渡してくる環境変数を優先します。ただしローカルでCLIとして動かす場合は環境変数を毎回 export するのが面倒なので、設定ファイルへのフォールバックも用意しました。

```python
def get_anthropic_key() -> str:
    key = os.environ.get("ANTHROPIC_API_KEY", "").strip()
    if key:
        return key
    cfg = _load_config_file()
    key = (cfg.get("anthropic_api_key") or cfg.get("api_key") or "").strip()
    if key:
        return key
    raise RuntimeError(_bilingual_missing_key("ANTHROPIC_API_KEY"))
```

設定ファイルの場所は、Windows なら `%APPDATA%`、macOS なら `~/Library/Application Support`、Linux なら XDG 準拠と、OSごとに分けています。利用者が「どこに置けばいいか」で迷わない構造にするのは、地味ながら採用率に直結する要素です。

### 任意のコスト追跡を差し込む

LLMサーバーを公開すると、利用者がどれくらいコストを溶かすかが気になります。とはいえOSS利用者全員に計測を強制したくはないので、「環境変数 `ENABLE_COST_TRACKING=1` を立てた人だけログを取る」というオプトイン方式にしました。

価格表は単純な dict として持ちます。

```python
PRICING: dict[str, dict[str, float]] = {
    "claude-sonnet-4-6": {"input": 3.00,  "output": 15.00},
    "claude-haiku-4-5":  {"input": 0.80,  "output": 4.00},
    "claude-opus-4-7":   {"input": 15.00, "output": 75.00},
    "gpt-4o":            {"input": 2.50,  "output": 10.00},
    "gpt-4o-mini":       {"input": 0.15,  "output": 0.60},
    # 単位は USD / 1M tokens
}
```

実際の追記処理（JSONL appender）は API 呼び出し直後に `usage` を1行JSONとして書き出すだけのシンプルな実装で、詳細はリポジトリの [reviewer.py](https://github.com/miharu8686/mutual-review-mcp/blob/main/src/mutual_review_mcp/reviewer.py) を参照してください。

## パッケージ化と PyPI 公開の要点

このセクションのゴールは、書いたMCPサーバーを `uvx 〜` の1コマンドで誰でも起動できる形に持っていく流れを掴むことです。

### pyproject.toml の最小構成

最近のPythonパッケージングは `pyproject.toml` 1ファイルで完結します。本記事の文脈で本質的なのは次の2ブロックです。

```toml
[project]
name = "mutual-review-mcp"
version = "0.1.1"
requires-python = ">=3.10"
dependencies = [
    "mcp>=1.0.0",
    "anthropic>=0.40.0",
    "openai>=1.50.0",
]

[project.scripts]
mutual-review-mcp = "mutual_review_mcp.server:main_sync"
mutual-review     = "mutual_review_mcp.cli:main"
```

ビルドバックエンドは hatchling を使っていますが、setuptools でも大差ありません。後から差し替えが効くよう、薄く保つのがコツです。

### console scripts で uvx 一発起動

`[project.scripts]` の効果は強力です。`mutual-review-mcp = "mutual_review_mcp.server:main_sync"` と書くと、pip install 後に `mutual-review-mcp` というコマンドが PATH に追加され、それを `uvx` が直接拾えるようになります。

エンドユーザーは pip install すら不要で、`uvx mutual-review-mcp` だけで PyPI から取得 → 隔離環境で実行までを完了できます。Claude Desktop 設定ファイル側もこれ前提で `"command": "uvx", "args": ["mutual-review-mcp"]` と書けば終わりです。

### 同一バージョンは再upload不可

PyPIの鉄則です。一度 `mutual-review-mcp-0.1.0` をアップロードしたら、内容を差し替えて同じ `0.1.0` を上げ直すことはできません（yankはできますが、空きバージョン番号として再利用もできません）。本記事の題材も、0.1.0 の `project.urls` に placeholder が残っていたのを 0.1.1 で修正することになりました。

実務上は、必ず先にTestPyPIへ上げて検証する流れを徹底するのが安全です。

### TestPyPI で必ず予行演習

TestPyPI は本番と独立したインデックスで、好きに上げ直して試せます。

```bash
# ビルド
python -m build

# メタデータ検証
twine check dist/*

# TestPyPI に上げて、クリーンな venv から落として動作確認
twine upload --repository testpypi dist/*
pip install --index-url https://test.pypi.org/simple/ \
            --extra-index-url https://pypi.org/simple/ mutual-review-mcp
```

ここで CLI / MCPプロトコルの両方を試して、問題なければ本番にアップロードします。

## Claude Code / Desktop からの呼び出し

ここまでで PyPI に上がっていれば、利用側の設定は一行〜数行で済みます。

### Claude Code: claude mcp add で完了

```bash
claude mcp add mutual-review -- uvx mutual-review-mcp
```

これでローカルの設定ファイルに登録され、以降のセッションで `/mcp` コマンドにこのサーバーが現れます。Anthropic と OpenAI のキーをシェルプロファイルで `export` しておけば、追加の設定は不要です。

### Claude Desktop: claude_desktop_config.json

GUI からは設定できないので、設定ファイルを直接書きます。場所は Windows なら `%APPDATA%/Claude/claude_desktop_config.json`、macOS なら `~/Library/Application Support/Claude/claude_desktop_config.json` です。

```json
{
  "mcpServers": {
    "mutual-review": {
      "command": "uvx",
      "args": ["mutual-review-mcp"],
      "env": {
        "ANTHROPIC_API_KEY": "sk-ant-...",
        "OPENAI_API_KEY": "sk-..."
      }
    }
  }
}
```

`env` ブロックでホストからサーバーへキーを渡しているので、シェル側で `export` する方法とは排他で考えられます（どちらか一方で十分です）。

### 動作確認: ツール一覧に出ているか

Claude Code なら `/mcp` で各サーバーの稼働状況とツール一覧が見られます。`review_file` / `review_code` / `review_diff` の3つが並んでいれば成功です。Claude Desktop の場合は、設定 → Developer → MCP Servers から状態を確認できます。

## 自分の業務にMCPを応用するヒント

MCPが書けるようになると、「LLMから何ができるか」の地図が一気に広がります。筆者が業務で実装してみたい、もしくは依頼を受けるとすぐ作れる3つの方向性を挙げます。

### アイデア1: 社内ドキュメント検索MCP

Notion / Confluence / Slack など、社内に散らばっている情報をLLMから引ける `search_docs(query, source)` のようなツールを提供します。

API トークンをサーバー側で持ち、検索結果のスニペットだけ LLM に渡す構成にすれば、ドキュメント本体は社内に置いたままで、Claude Code から「先週の会議の議事録、決定事項だけ抜き出して」と話しかける運用が可能になります。検索ロジックを差し替えるだけで、社内Wiki・ファイルサーバー・チケットシステムなど対象も柔軟に拡張できます。

### アイデア2: 定型業務自動化MCP

「議事録 → 要約 → Slack指定チャンネル投稿」のような決まった手順は、MCPサーバーに `post_meeting_summary(notes, channel)` という1ツールとして閉じ込めるのが向いています。

LLM 側のプロンプトに毎回手順を書く必要がなくなり、再現性も上がります。エラー処理やリトライをMCP側で吸収できるので、業務システムとLLMの境界がきれいに引けるのがメリットです。

### アイデア3: 業務固有レビューMCP（コードに限らない）

mutual-review-mcp と同じパターンは、コードに限らず適用できます。たとえば次のようなケースが考えられます。

- **契約書レビュー**: GPT が法務観点、Claude がビジネスインパクト観点で並列レビュー → 統合
- **デザインレビュー**: スクリーンショットを画像対応モデルに渡し、UX / アクセシビリティ / ブランド一貫性の3観点でレビュー
- **広告コピーレビュー**: 訴求力 / コンプライアンス / ブランドトーンの3レビューを統合

いずれも構造は同じで、ドメイン特化のシステムプロンプトと専用の入力スキーマを書けば、業務に直結したレビューツールが生まれます。

## まとめ

本記事では、MCPサーバーを書き、PyPIに公開し、Claude Code / Desktopから1コマンドで呼び出せるようにする流れを駆け足で見てきました。

ベースとした mutual-review-mcp は [github.com/miharu8686/mutual-review-mcp](https://github.com/miharu8686/mutual-review-mcp) で公開しています。コスト目安は haiku + gpt-4o-mini 構成で1ファイル（約500行）あたり約¥4、sonnet + gpt-4o 構成で約¥16です。手元のコードを試す程度なら気軽に使える価格帯です。

### シリーズ予定（執筆中）

- 第2回: mutual-review-mcp の設計詳解（執筆中）
- 第3回: WindowsでMCPサーバーを公開する際の落とし穴（執筆中）

### お問い合わせ

Claude Code 導入支援・カスタムMCP受託のご相談を承っています。

→ [お問い合わせフォーム](https://tally.so/r/aQad1v)

X (旧Twitter): [@miharu_tools](https://x.com/miharu_tools)（DMでもOK）
