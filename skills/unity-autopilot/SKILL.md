---
name: unity-autopilot
description: Unity MCP でゲームを自走（指定時間で計画→実装→仕上げ）で作るときに使う。引数なしで発動可能。発動時に playbook を現プロジェクトへコピー、ヒアリング→Planモード→ユーザー承認→自走実行までを一連で進める。スポット作業（バグ修正・1機能追加・質問応答等）には使わない。
---

# Unity Autopilot Skill

このスキルは Unity プロジェクト内で「自走でゲームを作る」フローを起動する。

**使用するツール**: Bash, Read, Write, Edit, AskUserQuestion, ExitPlanMode, TaskCreate / TaskUpdate, Agent (Explore / Plan), Unity MCP ツール全般

このスキルが発動したら、以下の手順を順守する。

## Step 1: 環境チェック + Playbook コピー

### 1.1 ソース存在確認
Bash で `[ -d "$HOME/UnityProject/unity-mcp-playbook" ]` を確認する。
**存在しない場合はユーザーに以下を伝えて中断する**（このスキルは続行不可）:

```
Playbook ソースが見つかりません。先に以下を実行してください:
  git clone https://github.com/LawrenceTurnerW/unity-mcp-playbook.git ~/UnityProject/unity-mcp-playbook
```

### 1.2 現プロジェクトへコピー
存在する場合、現在の cwd（Unity プロジェクトと想定）に playbook をコピーする:

```bash
mkdir -p ./playbook
cp "$HOME/UnityProject/unity-mcp-playbook/playbook.md" ./playbook/
cp -R "$HOME/UnityProject/unity-mcp-playbook/lessons" ./playbook/
```

無条件で上書き（再発動時も最新を反映）。

### 1.3 .gitignore へ追加
プロジェクト直下に `.gitignore` があれば `playbook/` が含まれているかを確認、なければ追記する:

```bash
if [ -f ./.gitignore ]; then
  grep -qxF 'playbook/' ./.gitignore || echo 'playbook/' >> ./.gitignore
fi
```

`.gitignore` 自体がない場合は、ユーザーの git 管理ポリシー次第なので何もしない。

### 1.4 CLAUDE.md にマーカー追記

`./CLAUDE.md` を確認:

- **存在しない**: ユーザーに「`CLAUDE.md` を新規作成して自走用のマーカーブロックを書きます。OK？」と AskUserQuestion で確認 → 承認後 Write で新規作成、下記マーカーブロックのみ書く
- **存在し、`<!-- unity-mcp-playbook:start -->` マーカーがある**: マーカー間を Edit で置換（最新版に更新、確認不要）
- **存在し、マーカーがない**: ユーザーに「既存の `CLAUDE.md` の末尾にマーカーブロックを追記します。OK？」と AskUserQuestion で確認 → 承認後 Edit で末尾追記

追記するマーカーブロック:

```markdown
<!-- unity-mcp-playbook:start -->
## Unity MCP 自走制作

このプロジェクトには `playbook/` に Unity MCP 自走制作のプレイブックがコピーされている。
自走依頼時は以下を参照:

- `playbook/playbook.md` — フロー本体
- `playbook/lessons/` — トピック別知見（必要時に Read）

スポット作業（バグ修正・1 機能追加・質問応答等）には参照不要。
<!-- unity-mcp-playbook:end -->
```

## Step 2: 開始時刻記録

```bash
date +%s > /tmp/game_start
```

## Step 3: ヒアリング（1 ターンで完結）

`AskUserQuestion` で以下 4 つをまとめて聞く:

1. **稼働時間**（分単位、例: 30 / 60 / 120）
2. **ジャンル**（TD / 2D アクション / 3D アクション / パズル / シミュ / AVN / 自由）
3. **基本思想**（フィードバック重視 / 戦略性重視 / 学習用 / プロトタイプ / 数値バランス重視 / 自由）
4. **既存リソース・制約**（持ち込み Asset / 既存シーン継続 / 特定の Package を使う / 環境固有事情 / なし）

## Step 4: 計画モード

### 通常モード（稼働時間 > 15 分）
1. `./playbook/playbook.md` を Read してフロー把握
2. EnterPlanMode
3. Explore agent 1-3 並列で現状把握（既存コード、Asset 構成、Package 有無）
4. Plan agent 1（複雑なら 2-3）で設計検証（ROI 評価・落とし穴予測）
5. Plan ファイル書く: Context / Approach / Critical Files / Time Budget / Risks / Verification
6. ExitPlanMode で承認待ち

### 短時間モード（稼働時間 ≤ 15 分）
1. `./playbook/playbook.md` を Read
2. **Explore / Plan agent は省略**
3. **Plan ファイルは書かない**、TaskCreate でフェーズだけ列挙
4. ユーザーに 1 行で計画提示 → 承認確認

## Step 5: 自走実行

ユーザー承認後、`./playbook/playbook.md` の Step 3 フェーズ実装サイクルに従う:

```
a. スクリプト一括生成 (batch_execute + manage_script)
b. コンパイル待ち + read_console
c. シーン構築
d. 配線 (SerializeField)
e. 小リファクタ（5 分の軽い確認、playbook §C 上半分のチェックリスト）
f. シーン save
g. (主要 Phase) Play 検証 + screenshot
h. 経過時間ログ
```

主要規則:
- **時間規律**: フェーズ単位は ±25% 許容、**全体は ±10% で着地**（厳密でなくていいが意識する）
- **60-70% 時点で専用リファクタフェーズ**（playbook §4、§C 全体を深掘り）。Step 3-e の小リファクタとは別物
- **80% 超過で新機能打ち切り**（playbook §6）
- 詰まったら `./playbook/lessons/<topic>.md` を Read して個別解決

## Step 6: 仕上げ

- 統合 Play 検証
- メモリ更新（プロジェクト memory ディレクトリ）
- 最終報告: 経過時間 / 達成機能 / 残課題 / 自己評価

---

## このスキルを使わないケース

以下は普通に作業する（このスキルを起動しない）:
- バグ修正・1 機能追加・質問応答などのスポット作業
- 5 分以内の小タスク

Unity 関連で詰まった場合、`./playbook/lessons/<topic>.md` を必要に応じて個別 Read してよい（スキル起動は不要）。
