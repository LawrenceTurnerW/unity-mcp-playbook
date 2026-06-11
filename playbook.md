# Unity MCP ゲーム自走制作プレイブック

ユーザーから「Unity MCP で N 時間自走でゲーム作って」「拡張して」と依頼されたら、まずこれを読む。ジャンル不問の汎用手順書。

知見（罠・コツ）は §A に **索引のみ** 配置、詳細は `lessons/<topic>.md` 配下にある（Single Source of Truth）。必要に応じて該当 lesson を Read する。

---

## 三原則

1. **時間規律で完走する**: 指定時間の **±10%** で着地。早く終わらない・大幅超過しない
2. **フェーズ末ごとに小リファクタ、全体の 60-70% 時点で専用リファクタフェーズを 1 回**
3. **UI / 入力 / グローバル状態の落とし穴は §A の索引から該当 lesson を読んで確認してから書く**

---

## 手順

### Step 0. セッション開始（〜2 分）

1. **開始時刻記録**: `date +%s > /tmp/game_start` を Bash で必ず実行
2. **プロジェクトメモリ読み込み**: `MEMORY.md` 経由でロード済みのはず。`project_*`, `unity-mcp-*` 系を頭に入れる
3. **環境確認**:
   - Unity Editor の `InteractionMode=NoThrottling` が設定済みか（未設定なら適用、§A1）
   - MCP サーバーの接続（`mcpforunity://instances`）
   - 開きたいシーンがあれば現在の active scene 確認
4. **ユーザー要求の整理**: ジャンル / 時間予算 / 自由度（仕様は Claude に任せる範囲）を頭で明確化

### Step 1. 計画（〜全体の 10-15%）

1. **Plan モードに入る**（EnterPlanMode）
2. **Explore agent 1-3 並列**で現状把握
   - 既存コード・Asset 構成・パッケージ有無を 300 words 以内で取得
   - Read 直接は避ける（Unity プロジェクトは無駄ファイルが多くコンテキストを食う）
3. **Plan agent 1（複雑なら 2-3）**で設計検証
   - ROI 評価 / 落とし穴予測 / 順序の妥当性をプロンプトに含める
   - 自信があるときでもサニティチェックとして必ず実行
4. **Plan ファイル書く**: Context / Approach / Critical Files / Time Budget / Risks / Verification を含める
5. **AskUserQuestion** は本当に詰まったときだけ。ユーザーは「自走で」と依頼している
6. **ExitPlanMode** で承認得て実装へ

短時間（〜15 分）の場合は Explore / Plan agent 省略、Plan ファイルは書かず TaskCreate のみで進む。

### Step 2. タスク分解と時間配分（〜2 分）

1. **TaskCreate** で各フェーズをタスク化（5-10 個推奨）
2. **時間バジェットを書き出す**: フェーズごとの想定時間を Plan ファイルの Time Budget セクションに
3. **マイルストーン**: 全体の 60-70% の時点で「専用リファクタフェーズ」を 1 つ必ず予約

### Step 3. フェーズ実装（繰り返し）

各フェーズは以下を **すべて完了** してから次へ。1 つでも飛ばすと負債になる。

```
a. スクリプト一括生成
   - batch_execute + manage_script (action=create) ※ "create_script" 名は通らない（§A2）
   - 関連スクリプトは 1 バッチでまとめる → ドメインリロード 1 回
b. コンパイル待ち + エラーチェック
   - refresh_unity (compile=request, wait_for_ready=true)
   - read_console (types=["error"], count=10)
c. シーン構築
   - manage_gameobject / manage_components で階層作成
   - prefab 化は manage_prefabs.create_from_gameobject、編集は modify_contents
d. 配線（SerializeField）
   - 単純なものは manage_components.set_property
   - 複雑なリスト / 多数フィールドは execute_code (codedom) + SerializedObject (§A3)
e. ⚠️ 小リファクタ（5 分の軽い確認、§C 上半分のチェックリスト）
f. シーン save (manage_scene.save)
g. 主要フェーズのみ Play 検証 + screenshot
h. 経過時間ログ
```

**所要時間の見積もり**：
- 新規スクリプト 5-10 本のフェーズ: 10-20 分
- シーン構築 + 配線中心のフェーズ: 5-15 分
- UI 構築中心のフェーズ: 10-20 分

### Step 4. 専用リファクタフェーズ（全体の 60-70% 時点、必須）

Step 3-e の小リファクタとは **別物**。Step 3-e が軽い確認なら、こちらはまとまった時間を取って深いリファクタを行う。スキップせず必ず実施。

実施項目（§C のチェックリスト全体を見ながら深掘り）:

1. **重複パターン抽出**: 3 箇所以上で同じことをやってたら共通化
2. **責務分割**: 1 クラスが「ロジック + UI + 入力」を持っていれば分離
3. **イベント購読の対称性**: OnEnable/OnDisable のペア、`subscribed` フラグの整合性
4. **グローバル状態の集約**: `Time.timeScale`, `Cursor.lockState`, `Application.targetFrameRate` を触る箇所が複数あれば 1 つに（§A4）
5. **API ファサード追加**: 外部から触られるコンポーネントには `Tower.Priority` / `Tower.KillCount` のような facade プロパティを用意
6. **using 整理 / マジックナンバー昇格 / 命名統一**

時間目安: 全体の **5-10%**（60 分予算なら 4-6 分）

### Step 5. 仕上げ（〜全体の 10%）

1. **統合 Play 検証**: 想定する遊び方を Play モードで 1 回通す。Screenshot を 1-2 枚撮る
2. **メモリ更新**:
   - 現プロジェクトの session ファイル新規 or 追記
   - 新発見の知見は `lessons/<topic>.md` に追記（本体）し、§A に 1 行索引を追加
   - MEMORY.md にリンク追加
3. **最終報告**: 経過時間 / 達成機能リスト / 残課題 / 自己評価。冗長にしない

### Step 6. 残時間の使い方（重要、§E 参照）

時間予算の 80% 超で未着手機能があれば打ち切り、リファクタ・検証・ドキュメントに切り替える。

時間が余ったら **その順番で**:
1. リファクタ深掘り（§C のチェックリスト + 重複削減）
2. Play でユーザー視点で 1 周遊んで（Game View で）気付いたバグを直す
3. 軽い polish（UI 調整 / 数値バランス / 音量）
4. メモリ更新を厚く（lessons/ に追記、§A に索引追加）
5. **新機能追加は最後の手段**。残 15% 未満では着手しない（コンパイル不能で時間切れリスク）

---

## ジャンル別 配慮

### Tower Defense / シミュレーション
- Waypoint path で 9 割解決。NavMesh は不要（§A8）
- 経済システム（gold/exp）はイベント駆動が向く

### 2D / 3D アクション
- 物理（Rigidbody）と独立移動を最初に決める
- カメラは追従 vs オービット vs 固定を Plan で確定
- 入力は Input System の Action Asset で抽象化推奨

### パズル / ボードゲーム
- セル / グリッドのデータ構造を最初に固める
- アニメーション中の入力ロックを忘れない（state machine）

### ストーリー / AVN
- ScriptableObject でシーン定義、データ駆動
- TMP 必須（リッチテキスト / フォント）→ 必要なら早めに Package 追加

### 共通
- **データは ScriptableObject**：Enemy / Tower / Item / Skill 等。Inspector で編集できて差分が見える
- **設定は SerializeField + [Tooltip]**：マジックナンバー直書きは避ける
- **音は手続き型からでも始められる**（§A7）

---

## §A 知見索引（詳細は lessons/ に）

§A は索引のみ。タイトル + 1 行教訓 + 詳細ファイルへのリンク。

| § | 教訓 | 詳細 |
|---|---|---|
| **A1** | Editor の NoThrottling 必須 — 非フォーカス時の play 停止を防ぐ | [tooling.md#play-モードとフォーカス重要な解決法](lessons/tooling.md#play-モードとフォーカス重要な解決法) |
| **A2** | MCP 経由のツール名・パラメータの罠 — batch_execute / LayerMask / 配列 SO / UI 構築 等 | [tooling.md](lessons/tooling.md) |
| **A3** | SerializeField 配線は execute_code + SerializedObject 一発 | [tooling.md#execute_code-の使いどころ](lessons/tooling.md#execute_code-の使いどころ) |
| **A4** | グローバル状態（timeScale 等）の Awake 競合 — reset は 1 controller に集約 | [tooling.md#timetimescale-の-awake-競合](lessons/tooling.md#timetimescale-の-awake-競合) |
| **A5** | UI と World 入力の競合 — `EventSystem.IsPointerOverGameObject()` で必ずガード | [tooling.md#ui-と-world-入力の競合](lessons/tooling.md#ui-と-world-入力の競合) |
| **A6** | Script Execution Order — singleton 購読は OnEnable + Start 二重で安全 | [tooling.md#script-execution-order](lessons/tooling.md#script-execution-order) |
| **A7** | 手続き型 AudioClip で素材ゼロから効果音 / BGM | [tooling.md#手続き型-audioclip-による効果音--bgm-生成](lessons/tooling.md#手続き型-audioclip-による効果音--bgm-生成) |
| **A8** | NavMesh vs Waypoint — 固定経路ゲームは Waypoint 一択 | [tooling.md#navmesh-vs-waypoint](lessons/tooling.md#navmesh-vs-waypoint) |
| **A9** | prefab 編集は `manage_prefabs.modify_contents` で headless 最速 | [tooling.md#prefab-編集](lessons/tooling.md#prefab-編集) |
| **A10** | World-Space ダメージ数字 — Canvas 1 つを共有、子 Text を spawn | [tooling.md#world-space-ダメージ数字簡易フローティングテキスト](lessons/tooling.md#world-space-ダメージ数字簡易フローティングテキスト) |
| **A11** | スクリーンシェイク — カメラ pivot に `Random.insideUnitSphere * mag` 時間減衰 | [tooling.md#スクリーンシェイク](lessons/tooling.md#スクリーンシェイク) |
| **A12** | EditorPrefs 隠しキーの発見法 — Unity 内部の IL を読んで ldstr を辿る | [tooling.md#editorprefs-キー特定の-reflection-技](lessons/tooling.md#editorprefs-キー特定の-reflection-技) |
| **anim** | `manage_animation` API リファレンス | [animation.md](lessons/animation.md) |

新しい知見が出たら **lessons/<topic>.md に本体を書き** → ここに索引行を追加する運用。

---

## §B ライブラリ判断

### Reactive Extensions
- **UniRx**: メンテ終了気味、新規導入は避ける
- **R3**（UniRx 後継）: ストリーム合成 / Throttle / Debounce / 自動 dispose が必要になったら検討
- **導入線**: イベント源 10+ / Throttle Debounce 複数箇所 / 連鎖が `event Action<>` で破綻
- **小〜中規模ゲームでは多くの場合不要**: `event Action<>` + `subscribed` フラグで十分

### 非同期
- **Unity Awaitable** (Unity 2023.1+ 標準): 軽量、依存追加なし。優先選択
- **UniTask**: より高機能、`UniTask.WaitUntil` 等が必要なら
- **Coroutine** は依然有効。WaitForSeconds で十分なケースで複雑化させない

### UI
- **uGUI (Legacy Text)**: 高速プロトの第一候補。TMP 未導入でも書ける
- **TMP**: 必要になったら Package 追加（リッチテキスト / カスタムフォント）
- **UI Toolkit**: 設定画面など複雑 UI で有利、ただし `manage_ui` (UXML/USS) でしか操作できない

### 物理 / アニメーション
- 単純な移動・回転は `transform` 操作で十分
- Rigidbody は必要になったら（衝突応答、力学）
- Animator は外部モーション流し込み時に。手作りアニメーションなら直接 transform で

---

## §C リファクタチェックリスト

**上半分**（Step 3-e の小リファクタで使う、5 分の軽い確認）:

- [ ] 1 クラス 200 行 / 1 メソッド 40 行 超えてないか
- [ ] マジックナンバーが `[SerializeField]` か SO に昇格されているか
- [ ] 命名のプレフィックス統一（Tower*, Enemy*, Wave*, UI* 等）
- [ ] `[Tooltip]` が SerializeField に付いているか
- [ ] using 重複なし

**下半分**（Step 4 の専用リファクタで深掘り、5-10% 時間使う）:

- [ ] 直接参照より event / interface を使えているか
- [ ] フォルダ階層が責務と一致しているか
- [ ] 一時追加して未使用になったコード・ファイルを削除したか
- [ ] OnEnable / OnDisable が対称か（イベント購読忘れの常套手口）
- [ ] グローバル状態（timeScale 等）を触るのが 1 箇所か
- [ ] singleton accessor のガードがあるか（`if (Instance == null) return`）
- [ ] 3 箇所以上で同じパターンが出てきたら共通化されているか

---

## §D 時間配分の目安

| 全体時間 | 計画 | 実装 | 専用リファクタ | 仕上げ |
|---|---|---|---|---|
| 30 分 | 4 分 | 18 分 | 3 分 | 5 分 |
| 60 分 | 8 分 | 36 分 | 6 分 | 10 分 |
| 120 分 | 15 分 | 75 分 | 12 分 | 18 分 |

許容範囲: **フェーズ単位は ±25% 許容、全体は ±10% で着地**。厳密でなくていいが意識する。専用リファクタを削るのは禁止。

---

## §E 自走の心構え

- **「完成」と「予算消化」はイコールではない**。指定時間が余ったらリファクタ・検証・ドキュメントに使う。新機能を闇雲に積まない
- **UI 入力ガードは新規 Behaviour 書く前に §A5 を確認** する習慣をつける
- **グローバル状態は 1 箇所集約**（§A4）
- **同じ罠を踏み続けない**: 詰まったら lessons/ に追記してから次へ。後で書こうとすると忘れる
- **Plan モードを軽視しない**: Explore と Plan agent は時間の無駄に見えるが、後段の手戻りを大幅に減らす

---

## 次回開始テンプレート

```
1. このプレイブックを読む
2. Bash: date +%s > /tmp/game_start
3. EditorPrefs InteractionMode=1 確認（§A1）
4. EnterPlanMode → Explore 1-3 並列 → Plan agent → ExitPlanMode
5. TaskCreate でフェーズ分解、Time Budget は §D
6. フェーズ実装ループ（Step 3 のチェックリスト遵守）
7. 60-70% 時点で専用リファクタ（§C 下半分 + 重複削減）
8. 80% 時点で新機能打ち切り判定
9. 仕上げ + メモリ更新 + 報告
```
