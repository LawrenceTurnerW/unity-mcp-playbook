# `manage_animation` API リファレンス

MCP for Unity v9.7.x の `manage_animation` ツールで AnimationClip と AnimatorController を作る際のパラメータ仕様。Unity 6 で動作確認済み。

## `clip_create`

`clip_path` 必須、`properties` に `loop` (bool), `frameRate` (int)。デフォルト frameRate=60。

```
manage_animation(
  action="clip_create",
  clip_path="Assets/Animations/SpinY.anim",
  properties={"loop": true, "frameRate": 60}
)
```

## `clip_add_curve`

`clip_path` + `properties: {type, relativePath, propertyPath, keys}`。

注意点:

- パラメータ名は **`propertyPath`** （`propertyName` ではない）。誤ると `'propertyPath' is required` エラー
- Transform の Y軸回転は `propertyPath: "localEulerAnglesRaw.y"` を使う（Quaternion 経由ではなく、360 度を超える回転も扱える）
- `relativePath: ""` でルート GameObject
- `keys`: `[{"time": 0, "value": 0}, {"time": 2, "value": 360}, ...]`

```
manage_animation(
  action="clip_add_curve",
  clip_path="Assets/Animations/SpinY.anim",
  properties={
    "type": "UnityEngine.Transform",
    "relativePath": "",
    "propertyPath": "localEulerAnglesRaw.y",
    "keys": [
      {"time": 0, "value": 0},
      {"time": 1, "value": 180},
      {"time": 2, "value": 360}
    ]
  }
)
```

## `controller_create`

`controller_path` のみで OK。空の Base Layer ができる。

```
manage_animation(
  action="controller_create",
  controller_path="Assets/Animators/SpinningCube.controller"
)
```

## `controller_add_state`

`properties: {stateName, clipPath, setAsDefault}`。

- レスポンスの `isDefault` が false でも、レイヤー内に他に state がなければ Unity 側では実質 default になる（`controller_get_info` で確認可能）
- **`controller_set_default_state` というアクションは存在しない**。明示的に default を切り替える手段は未確認

```
manage_animation(
  action="controller_add_state",
  controller_path="Assets/Animators/SpinningCube.controller",
  properties={
    "stateName": "Spin",
    "clipPath": "Assets/Animations/SpinY.anim",
    "setAsDefault": true
  }
)
```

## `controller_assign`

`controller_path` + `target` + `search_method` で、対象 GameObject に Animator コンポーネントを自動追加してコントローラーを割り当てる（既存 Animator がなくても新規アタッチされる）。

```
manage_animation(
  action="controller_assign",
  controller_path="Assets/Animators/SpinningCube.controller",
  target="SpinningCube",
  search_method="by_name"
)
```

## 典型フロー

アニメーションクリップを 1 から作って GameObject に割り当てる基本的な流れ:

1. `clip_create` でクリップ生成
2. `clip_add_curve` でキーフレーム追加（複数プロパティなら繰り返し）
3. `controller_create` でコントローラ生成
4. `controller_add_state` でクリップを state として登録
5. `controller_assign` で GameObject に割り当て（Animator 自動追加）

Humanoid Avatar のリターゲット、Blend Tree、IK 等の高度なケースは別途検証が必要。
