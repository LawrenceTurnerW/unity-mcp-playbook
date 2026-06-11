# Unity MCP ツール経由のハマりどころ集

MCP for Unity v9.7.x + Unity 6 (URP) における罠と回避策。playbook §A2 の詳細版。

## バッチ実行時のツール名

`batch_execute` で `create_script` を使うと `"Unknown command type"` エラー。**`manage_script` (action=create) を使う**。`manage_script` 全般が batch_execute 内で利用可能。

## スクリプト編集

- `manage_script.delete` → 削除後、シーン内のコンポーネント参照は意外と保持される（同名同 namespace で再作成すれば re-link 成功）
- `script_apply_edits.replace_method` は便利だが、置換テキストの直前直後インデントに敏感
- Auto Mode classifier は破壊的削除をブロックすることがある。代わりに `Edit` / `script_apply_edits` で書き換えるか、`Write` で上書きするのが安全

## コンポーネント追加と SerializeField

- `manage_components add` で追加した直後、SerializeField の `=~0` のような C# field initializer は Unity の serializer に無視されることがある。**Awake で `if (mask.value == 0)` フォールバックを書く**のが安全
- **LayerMask の set_property は int 直渡しが失敗する**（`Error converting value 1024 to type 'UnityEngine.LayerMask'`）。コード側で動的決定する方が確実

## マテリアル割り当ての target 指定

`manage_material assign_material_to_renderer` で `target` に `"Parent/Child"` 形式のパスを渡すと **"Could not find target GameObject"** で失敗する。**instanceID を文字列で渡し `search_method: "by_id"` 指定**するか、子オブジェクトの名前で by_name 検索する。

## Prefab 編集

- `manage_prefabs create_from_gameobject` でシーン内インスタンスを Prefab 化すると、インスタンスは Prefab にリンクされた状態で残る（`wasUnlinked: false`）
- `manage_prefabs modify_contents` の `components_to_add` パラメータで Prefab に直接コンポーネント追加可能（プレハブステージを開く必要なし）
- レイヤー伝播は手動。子オブジェクトに親レイヤーを継承させたい場合は `GetComponentsInChildren<Transform>` でループして明示的に設定（execute_code で書ける）

## ScriptableObject 配列の操作

- `entries.Array.size` は **"Unsupported SerializedPropertyType: ArraySize"** で直接設定不可
- が、`entries.Array.data[N]` への set は配列を自動拡張するため、size を気にせず `[0], [1], [2]` 順で書けば動く
- ネストした struct (`entries.Array.data[0].count` 等) も問題なく動作

## UI 構築

- `manage_ui` は UI Toolkit (UXML/USS) 専用。uGUI Canvas/Button/Text は `manage_gameobject` + `manage_components` 経由だと手数が多すぎる
- **代わりに `execute_code (compiler=codedom)` で C# を直接実行するのが圧倒的に速い**。`using` 句は不要で、すべて完全修飾名（`UnityEngine.UI.Text`, `UnityEditor.SerializedObject` 等）で書く
- Roslyn は標準では未導入のため `codedom` 一択（C# 6 のみ。`=>` 式は OK だが `using` ディレクティブ NG）
- **Button.onClick は AddListener で追加した listener が serialize されない**ため、編集時に直接 AddListener しても再生時には消える。専用 MonoBehaviour（`TowerBuildButton` 等）の OnEnable で AddListener する設計が必須

## 手続き型 AudioClip による効果音 / BGM 生成

素材無しでゲーム音を作る方法:

- `AudioClip.Create(name, samples, channels, sampleRate, false)` + `clip.SetData(float[], 0)` で動的生成。AudioListener と AudioSource があれば即再生可
- 効果音は `Mathf.Sin(2πf × t/SR) × Mathf.Exp(-t × decay)` で「ピューン」「ボン」が作れる。ノイズは `System.Random.NextDouble()*2-1`
- BGM ループは `audioSource.loop = true` で。bass + lead + pad の 3 トラック合成（各 `Mathf.Sin` を加算）で 8 拍の small ループが OK
- Awake で AudioClip をキャッシュして `PlayOneShot(clip, volume)` で繰り返し再生（GC 圧無し）
- PlayerPrefs で `MusicVol` / `SfxVol` を保存して次回ロード

## World-Space ダメージ数字（簡易フローティングテキスト）

- シーンに **1 つの WorldSpace Canvas** を生成し、ダメージ毎に Text を子 spawn。各 Text に Canvas を持たせると drawcall 爆発
- `LateUpdate` で `transform.position += Vector3.up * speed * dt`、`Color.a = 1 - t*t` で速い fade
- Camera に向いた billboard は `transform.rotation = Camera.main.transform.rotation`
- スケール: `localScale = Vector3.one * 0.012f` (1 unit = 100 text units 相当)

## Time.timeScale の Awake 競合

`Time.timeScale` 等のグローバル状態を **複数の MonoBehaviour の Awake で** 触ると、Awake 順序は非決定なので最後の上書きが勝つ。意図しない値で確定して再現困難なバグになる。

例: モーダルパネルが Awake で 0 に設定したのに、別のマネージャーが Awake で 1 にリセットしていて、結果ゲームが普通に進行してしまう。

**ルール**:
- グローバル状態（`Time.timeScale`, `Cursor.lockState`, `Application.targetFrameRate` 等）の reset を Awake で行うコンポーネントは **1 つだけ** に絞る
- 他のコンポーネントは「明示的なトリガで状態変更」のみ。Awake で勝手に触らない

## EditorPrefs キー特定の reflection 技

隠し EditorPref の key を探すには、対応する Preferences UI コードの IL を `MethodInfo.GetMethodBody().GetILAsByteArray()` で取得し、opcode 0x72 (ldstr) を辿る:

```csharp
for (int i = 0; i < il.Length; i++) {
  if (il[i] == 0x72 && i + 4 < il.Length) {
    int token = il[i+1] | (il[i+2]<<8) | (il[i+3]<<16) | (il[i+4]<<24);
    string s = method.Module.ResolveString(token);
    // ldstr "..."
  }
}
```

この手法で `UnityEditor.PreferencesProvider.DrawInteractionModeOptions` の中の `"InteractionMode"` 等、ドキュメント化されていない隠し EditorPref キーを発見できる。

## Play モードとフォーカス（重要な解決法）

- Unity Editor は **デフォルトで非フォーカス時に強烈な throttle がかかり、`playmode_transition` フェーズが 1 分以上続く** ことがある
- `Application.runInBackground = true` は実行ファイル向けで Editor の throttle には効かない
- **解決**: EditorPrefs キー `InteractionMode` を 1 (NoThrottling) に設定すると、非フォーカスでもフルレート稼働になる:
  ```csharp
  UnityEditor.EditorPrefs.SetInt("InteractionMode", 1); // 0=Default, 1=NoThrottling, 2=MonitorRefreshRate, 3=Custom
  typeof(UnityEditor.EditorApplication)
      .GetMethod("UpdateInteractionModeSettings", System.Reflection.BindingFlags.NonPublic | System.Reflection.BindingFlags.Static)
      .Invoke(null, null);
  ```
- 関連キー: `ApplicationIdleTime` (Custom モード用、ms), `InputMaxProcessTime`
- enum 定義は `UnityEditor.PreferencesProvider+InteractionMode` で確認可能（リフレクションで取得）
- これにより MCP 経由で Play モードの動作検証が完全に自走可能になる

## execute_code の使いどころ

- 多数の GameObject / コンポーネントを一気に組み立てる（UI 構築や階層変更）
- `UnityEditor.SerializedObject` 経由の SerializeField 設定（手作業より速い）
- Prefab を `PrefabUtility.LoadPrefabContents` / `SaveAsPrefabAsset` で開いて編集
- **`EditorSceneManager.MarkSceneDirty` を必ず呼ぶ**（変更が保存されない）

## read_console のノイズ

URP では `Ignoring depth surface load action as it is memoryless` という warning が常時出る（URP の memoryless attachment 関連、無害）。MCP の WebSocket warning も常時出る。**実害ないので無視 OK**。

## レイヤー追加と raycast マスク

raycast / OverlapSphere で対象を絞り込みたいときはレイヤーを早めに定義する。各レイヤーは `manage_editor add_layer` で追加 → Unity が slot 8 以降に自動割り当て。

ジャンルに応じて慣例的な分け方:
- アクション/シューター: Player / Enemy / Projectile / Environment / Trigger
- シミュレーション/タワーディフェンス: Ground / Buildable / Enemy / Friendly
- パズル: Cell / Piece / UI3D

レイヤーマスクの int 値が必要なときは `1 << LayerMask.NameToLayer("Enemy")` で動的に作る（マスクをハードコードしない）。

## UI と World 入力の競合

uGUI ボタンの上で World に対する Raycast（クリック起動の操作）が動いて誤発火する。例: ボタンと地面クリック等、World との同時入力で意図しない動作。

新規にマウスクリック等で World に作用する Behaviour を書いたら、最初にこのガードを入れる:

```csharp
if (UnityEngine.EventSystems.EventSystem.current != null
    && UnityEngine.EventSystems.EventSystem.current.IsPointerOverGameObject()) return;
```

対象: マウスクリックで地面 / オブジェクトに対する操作を発火するすべての Update。

## Script Execution Order

同フレームの Awake は GameObject 列挙順で非決定。確実な順序が必要なら:
- Project Settings > Script Execution Order で明示
- もしくは `Instance != null` ガード + `Start` で再試行（`subscribed` フラグパターン）

Singleton にイベント購読する UI 系は **OnEnable で 1 回 + Start で再試行** が安全。

## NavMesh vs Waypoint

- 固定経路ゲーム（タワーディフェンス、レール式 STG 等）は Waypoint 一択。NavMeshAgent の avoidance / stoppingDistance が非決定挙動を生む
- NavMesh が活きるのは動的経路探索が必要なケース（RTS, 探索型 action, AI 群衆）

## スクリーンシェイク

- カメラ pivot に `Random.insideUnitSphere * magnitude` を加算、時間減衰
- magnitude 0.15-0.7、duration 0.1-0.5 が体感ベスト
- ボス級・大型 AOE は通常の 4-5x
