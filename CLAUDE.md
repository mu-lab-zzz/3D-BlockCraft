# 3D-BlockCraft — CLAUDE.md

## プロジェクト概要

Three.js r128 + CANNON-es を使ったシングルファイル PWA のブロック組み立て＆サバイバルゲーム。
ソースは **`index.html`** 1ファイルに全て含まれる（CSS・JS・HTML一体型）。

- **ビルドモード**: LEGOライクなブロックでビークルを組み立てる
- **サバイバルモード**: 組み立てたビークルで走り回り、地形上のオブジェクトを採掘・収集

---

## 技術スタック

| 項目 | 内容 |
|------|------|
| レンダリング | Three.js r128 (CDN) |
| 物理 | CANNON-es (インライン) |
| PWA | `sw.js` でオフラインキャッシュ (`blockcraft-v*`) |
| 永続化 | localStorage |
| 対象 | モバイル優先（タッチジョイスティック）、PC対応 |

---

## 座標系・グリッド

- **ハーフグリッド**: Y軸は `originHY`（半グリッド単位）で管理。ワールドY = `originHY * 0.5`
- **薄板ブロック** は高さ 0.5 (originHY += 1)、標準ブロックは高さ 1.0 (originHY += 2)
- `occupiedGrid`: `"x,hy,z"` をキーとする Set でブロック重複チェック
- ビルドグリッドサイズ: `GRID_SIZE = 50`

---

## 主要定数

```javascript
WHEEL_GROUND_OFFSET = 0.83   // kinematic時の地上高
TC_SIZE   = 256              // チャンクサイズ (m)
TC_RADIUS = 2                // ロード半径 (5×5チャンク)
RMAX      = 25               // 岩の最大数/チャンク
DRILL_RANGE   = 2.4          // ドリル破壊検知半径 (blocks) ※岩は+岩半径
PICKUP_RANGE  = 4.0          // アイテム自動回収半径
BASE_POS      = { x: 12.5, z: 11.0 }   // 拠点センター (スポーン付近)
BASE_RANGE    = 12.0         // 拠点自動収納&ボタン表示半径
COCKPIT_CARGO = { cockpit1x1:5, cockpit2x2:10, cockpit2x3:15, cockpit2x4:20 }
```

---

## ブロックタイプ

### タイヤ系
| type | 説明 |
|------|------|
| `wheel` | 標準タイヤ |
| `wheel3` | オフロード径3 |
| `wheel5` | スポーツ径5 |
| `wheel5t2` | ヘビーデューティ径5厚（アクスル2ユニット） |

`isLargeWheelType(type)` → wheel3/wheel5/wheel5t2

### 円柱系
`cyl`, `cylp`(薄), `cyl05`, `cyl05p`, `cyl2`〜`cyl5`(大型), `cyl2p`〜`cyl5p`(大型薄)

`isCylinderType(type)` → 全円柱判定  
`getLargeCylSize(type)` → 2〜5の数値 or null

### コクピット系
`cockpit1x1`, `cockpit2x2`, `cockpit2x3`, `cockpit2x4`

`isCockpitType(type)` → コクピット判定  
積載量は `COCKPIT_CARGO[type]` で取得

### その他
`connector` (接続面), `drill` (採掘ドリル), 通常ブロック各種

---

## ブロックメッシュ生成

```javascript
createBlockMesh(type, color, rot, tilt)   // 全タイプ振り分け
createWheelMesh(color, rot)
createLargeWheelMesh(type, color, rot)
createCylinderMesh(type, color, tilt)
createConnectorMesh(color, rot)
createDrillMesh(color, rot)               // +Z方向にビットが突出
createCockpitMesh(type, color, rot)
```

**ドリルの前方向**: ローカル +Z。`new THREE.Vector3(0,0,1).applyQuaternion(group.quaternion)` でワールド前方を取得。

### プレビューメッシュ
```javascript
createPreviewMesh()                       // 現在選択タイプに応じて切り替え
createWheelPreviewMesh(rot)
createLargeWheelPreviewMesh(type, rot)    // ★ 追加済み
createCylinderPreviewMesh(type, tilt)
createConnectorPreviewMesh(rot)
createDrillPreviewMesh(rot)               // ★ 追加済み
createCockpitPreviewMesh(type, rot)
updatePreviewMesh()                       // タイプ・回転変更時に呼び出す
```

---

## サバイバルモード

### 起動・終了
```javascript
enterSurvivalMode()   // ビークル物理構築、チャンク表示、拠点インベントリ初期化
exitSurvivalMode()    // 物理解放、ブロック位置リセット、各種状態保存
```

### メインループ
```javascript
updateSurvival(now)
  → updateVehicleTransform()       // 物理/kinematic でブロック位置更新
  → checkPickup()                  // ドロップアイテム自動回収
  → checkDrill(dt)                 // ドリルによる地形破壊
  → checkBaseRange()               // 拠点範囲チェック・自動収納
  → _updateTerrainChunks(x, z)    // チャンクストリーミング
  → updateSurvivalCamera()
  → updateBiomeAtmosphere(x, z)
```

### 車両物理
```javascript
buildVehiclePhysics()    // CANNON-es RaycastVehicle 構築
teardownVehiclePhysics() // 物理解放
updateVehicleTransform() // 毎フレーム: chassisBody位置→ブロック群に適用
```

`vehicleBlocksData[i] = { group, localPos, origRotY, ... }`

物理なしの場合は kinematic fallback（`chassisBody === null`）。

### 採掘・インベントリ
```javascript
// グローバル状態
const inventory        = { stone: 0, wood: 0 }   // 車両インベントリ
const baseInventory    = { stone: 0, wood: 0 }   // 拠点インベントリ
let totalCargoCapacity = 10                       // コクピット積載量合計
const droppedItems     = []                       // { mesh, type }
const destroyedTerrain = new Set()               // "cx,cz|rock|i" or "|tree|i"

destroyTerrainObject(chunkKey, 'rock'|'tree', idx)  // 破壊＋ドロップ
spawnDroppedItem(type, wx, wy, wz)                  // ドロップアイテム生成
checkPickup()                                       // 範囲内アイテム回収
checkDrill(dt)                                      // ドリルで前方破壊
updateInventoryHUD()                                // HUD更新
```

**岩のドリル判定**: 岩は物理ボディが大きいため `effR = DRILL_RANGE + rp.r`（岩半径）で判定。`rp.r = s * max(sx,sz) * 0.7` として `rockPositions` に保存済み。

### 拠点インベントリ
```javascript
checkBaseRange()          // 毎フレーム: 範囲内なら autoStoreToBase() + ボタン表示
autoStoreToBase()         // 車両インベントリを拠点に移して保存
openBaseInventory()       // パネル表示
closeBaseInventory()      // パネル非表示
updateBaseInventoryUI()   // stone/wood/total 表示更新
saveBaseInventory()       / localStorage 'blockcraft_base_inv'
loadBaseInventory()
```

---

## チャンク地形

```javascript
_buildTerrainChunk(cx, cz)     // 地形メッシュ + 岩/木/サボテン生成
_updateTerrainChunks(px, pz)   // ロード/アンロード
_unloadTerrainChunk(key)
```

`terrainChunkMap` (Map): キー = `"cx,cz"` (`_tcKey(cx,cz)`)

チャンクデータ構造:
```javascript
{
  mesh, physicsBody,
  objects, physicsBodies,
  rockMesh: InstancedMesh,
  rockPositions: [{ wx, wy, wz, r }],   // r = XZ半径
  rockPhysBodies: [],
  treeMeshes: [],
  treePositions: [{ wx, wy, wz }],
}
```

岩の破壊: `rockMesh.setMatrixAt(idx, scale(0,0,0))` + `instanceMatrix.needsUpdate = true`  
木の破壊: `treeMeshes[idx].visible = false`

---

## ワールドブロック (建設モード外)

```javascript
placeWorldBlock(wx, wy, wz, color)   // サバイバル世界に永続ブロック設置
saveWorldBlocks()                    // localStorage 'block_craft_world'
loadWorldBlocks()
buildSpawnShop()                     // スポーン地点の拠点ショップ生成 (BX=8, BZ=8, 9×7)
```

`worldBlockMap`: Map キー `"wx,wy,wz"` → Three.js Group

---

## localStorage キー

| キー | 内容 |
|------|------|
| `block_craft_autosave` | ビルドモード自動セーブ |
| `block_craft_slots` | ビルドモードセーブスロット |
| `block_craft_survival` | サバイバルオートセーブ |
| `block_craft_survival_slots` | サバイバルセーブスロット |
| `block_craft_world` | ワールドブロック |
| `blockcraft_destroyed` | 破壊済み地形オブジェクト Set |
| `blockcraft_base_inv` | 拠点インベントリ |

---

## 開発ブランチ運用

- 作業ブランチ: `claude/new-session-XXXXXX`
- マージ先: `main`
- キャッシュバージョン (`sw.js` の `CACHE_NAME`) とビルド文字列 (`index.html` の `footer-version`) を毎コミット更新する
