# Unity MCP Self-Drive Playbook

Claude が **Unity MCP** を使って **自走でゲームを作る** ときの手順書と知見集。

「N 時間自走で Unity ゲーム作って」と依頼されたときに、Claude がさっと参照して走り出せる形で整理している。

## このリポジトリの使い方

### セットアップ（初回 1 回）

```bash
git clone https://github.com/LawrenceTurnerW/unity-mcp-playbook.git ~/UnityProject/unity-mcp-playbook
mkdir -p ~/.claude/skills
ln -s ~/UnityProject/unity-mcp-playbook/skills/unity-mcp-setup ~/.claude/skills/unity-mcp-setup
ln -s ~/UnityProject/unity-mcp-playbook/skills/unity-autopilot ~/.claude/skills/unity-autopilot
```

Claude Code を再起動すれば `/unity-mcp-setup` と `/unity-autopilot` が使える状態に。

### 新しい Unity プロジェクトを始めるとき

```
/unity-mcp-setup
```
で MCP の半自動セットアップ → 案内に従って Unity Editor 起動 + Claude Code 再起動

```
/unity-autopilot
```
で自走ゲーム制作スタート

### 構成

- **[playbook.md](playbook.md)** — メインの手順書。最初に読む。三原則・Step 0-6・ジャンル別配慮・知見・チェックリスト・時間配分・次回開始テンプレート
- **[lessons/](lessons/)** — Unity MCP / Unity Editor のハマりどころ詳解。playbook §A 索引の本体
  - `tooling.md` — MCP ツール経由でのコード生成・配線・debug の罠と回避策（Awake 競合 / UI 入力ガード / NoThrottling / 手続きオーディオ等を含む Single Source of Truth）
  - `animation.md` — `manage_animation` API リファレンス
- **[skills/](skills/)** — Claude Code Skill 定義（クラウド管理対象）
  - `unity-mcp-setup/SKILL.md` — `/unity-mcp-setup` 新規 Unity プロジェクトへの MCP for Unity 半自動セットアップ
  - `unity-autopilot/SKILL.md` — `/unity-autopilot` 自走モード（指定時間で計画→実装→仕上げ）

## 知見の追記運用

新しい罠を踏んだら **その場で playbook の §A か lessons/ に追記**。後で整理するより忘れる前に書き留めるのが優先。汎用的に使える形（特定プロジェクトの事例ではなく、一般的なパターンとして）で書く。

## 前提環境

- Unity 6.x (6000.x) + URP
- MCP for Unity v9.7.x（CoplayDev/unity-mcp）
- Claude Code (Claude Opus 4.7 1M 推奨)
- macOS / Windows（手順は macOS で検証）

## License

WTFPL 相当の自由運用。
