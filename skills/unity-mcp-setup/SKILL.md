---
name: unity-mcp-setup
description: 新規 Unity プロジェクトに MCP for Unity を導入する半自動セットアップ。引数なしで発動。Unity Package 追加と Claude Code 側 MCP 登録（user scope）を自動化し、手動必須の手順（Unity Editor 起動 / MCP Server 確認 / Claude Code 再起動）はチェックリストで案内する。新規プロジェクトの初回セットアップに使う。
---

# Unity MCP Setup Skill

新規 Unity プロジェクトに MCP for Unity をセットアップする半自動スキル。Unity Package 追加と Claude Code 側 MCP 登録は自動、Unity Editor 操作とプロセス再起動は手動チェックリストで案内する。

**使用するツール**: Bash, Read, Edit, AskUserQuestion

## Step 1: 環境チェック

Bash で cwd が Unity プロジェクトか確認:

```bash
[ -f ./Packages/manifest.json ] && [ -d ./ProjectSettings ] && echo "ok" || echo "ng"
```

`ng` が出たら以下を伝えて中断:

```
ここは Unity プロジェクトではないようです（Packages/manifest.json または ProjectSettings/ が見つかりません）。
cd で Unity プロジェクトのルートに移動してから /unity-mcp-setup を再実行してください。
```

## Step 2: Unity Package 追加（自動）

`./Packages/manifest.json` を Read。`dependencies` 内に `"com.coplaydev.unity-mcp"` キーがあるか確認。

- **既にある場合**: 「MCP for Unity package は既に導入済みです」と報告してこの Step をスキップ
- **ない場合**: Edit で以下のエントリを `dependencies` に追加:

  ```json
  "com.coplaydev.unity-mcp": "https://github.com/CoplayDev/unity-mcp.git?path=/MCPForUnity#main"
  ```

  既存の `dependencies` 内の他エントリ（例: `"com.unity.ai.navigation"`）の隣に挿入。JSON の構文（カンマ）を壊さないよう注意。

## Step 3: Claude Code 側 MCP 登録（自動、user scope）

### 3.1 現在の登録状況確認

```bash
claude mcp list 2>&1 | grep -i unitymcp || echo "not_registered"
```

加えて、`~/.claude.json` を Python ワンライナで読んで scope を判定:

```bash
python3 -c "
import json
with open('$HOME/.claude.json') as f: d = json.load(f)
print('user_scope:', 'UnityMCP' in d.get('mcpServers', {}))
for p, cfg in d.get('projects', {}).items():
    if 'UnityMCP' in (cfg.get('mcpServers') or {}):
        print(f'local_scope:{p}')
"
```

### 3.2 ケース別アクション

| 現状 | アクション |
|---|---|
| user scope に登録済み | 「user scope に既に登録済み」と報告してスキップ |
| local scope（特定プロジェクト）に登録済み | ユーザーに確認の上、`claude mcp remove UnityMCP` を該当プロジェクトの cwd で実行し、その後 user scope に追加（次行） |
| 未登録 | 直接 user scope に追加 |

user scope への追加コマンド:

```bash
claude mcp add --scope user --transport http UnityMCP http://127.0.0.1:8080/mcp
```

local scope を削除して user に移行する場合、ユーザーには `AskUserQuestion` で:

```
local scope（プロジェクト紐付け）に UnityMCP の登録が見つかりました:
  <該当プロジェクトパス>

user scope（全プロジェクト共通）に統一しますか？
- はい: local scope を削除し user scope に登録（重複解消）
- いいえ: 現状維持（user scope への登録だけ行う）
```

## Step 4: 手動アクションのチェックリスト表示

自動部分が終わったら、以下を **そのまま表示** してユーザーに案内する:

```
✅ 自動セットアップ完了。残りの手動手順:

1. Unity Hub または Unity Editor で本プロジェクトを開く
   → 初回は MCP for Unity package の import に 30 秒〜1 分

2. Window > Package Manager で "MCP for Unity" の取り込み完了を確認

3. Window > MCP for Unity を開き、HTTP Server が
   http://127.0.0.1:8080 で動作中であることを確認
   （起動してなければウィンドウ内のボタンで起動）

4. Claude Code を一度終了して再起動
   → user scope の MCP 設定はセッション開始時に読み込まれるため

5. 新セッションで動作確認:
   - "claude mcp list" で UnityMCP が Connected になっているか
   - もしくは Claude に「mcpforunity://instances を読んで」と頼んで
     本プロジェクトの Unity インスタンスが表示されれば OK

これらが完了したら /unity-autopilot で自走開始可能。
```

## このスキルを使わないケース

- 既にセットアップ済みのプロジェクト（このスキルは冪等なので発動しても害はないが、不要）
- Unity プロジェクトではないディレクトリ（Step 1 で中断する）
