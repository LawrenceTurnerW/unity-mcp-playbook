# Unity MCP Self-Drive Playbook

Claude が **Unity MCP** を使って **自走でゲームを作る** ときの手順書と知見集。

「N 時間自走で Unity ゲーム作って」と依頼されたときに、Claude がさっと参照して走り出せる形で整理している。

## このリポジトリの使い方

### Claude に渡すとき

```
# 既に clone してあるなら
@playbook.md を読んで、これから N 時間 Unity MCP で自走してゲームを作る

# まだ clone してないなら（最新を取得）
git clone https://github.com/LawrenceTurnerW/unity-mcp-playbook.git ~/UnityProject/unity-mcp-playbook
# その後、@~/unity-mcp-playbook/playbook.md を読んでもらう
```

### 構成

- **[playbook.md](playbook.md)** — メインの手順書。最初に読む。三原則・Step 0-6・ジャンル別配慮・知見・チェックリスト・時間配分・次回開始テンプレート
- **[lessons/](lessons/)** — Unity MCP / Unity Editor のハマりどころ詳解。playbook §A 索引の本体
  - `tooling.md` — MCP ツール経由でのコード生成・配線・debug の罠と回避策（Awake 競合 / UI 入力ガード / NoThrottling / 手続きオーディオ等を含む Single Source of Truth）
  - `animation.md` — `manage_animation` API リファレンス
- **[skills/](skills/)** — Claude Code Skill 定義（クラウド管理対象）
  - `unity-autopilot/SKILL.md` — `/unity-autopilot` で発動する自走モードスキル

## 知見の追記運用

新しい罠を踏んだら **その場で playbook の §A か lessons/ に追記**。後で整理するより忘れる前に書き留めるのが優先。汎用的に使える形（特定プロジェクトの事例ではなく、一般的なパターンとして）で書く。

## 前提環境

- Unity 6.x (6000.x) + URP
- MCP for Unity v9.7.x（CoplayDev/unity-mcp）
- Claude Code (Claude Opus 4.7 1M 推奨)
- macOS / Windows（手順は macOS で検証）

## License

WTFPL 相当の自由運用。
