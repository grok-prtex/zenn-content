---
title: "cascadio で STEP を glTF にするときの単位・許容差・B-rep 情報（0.1.1 で確認）"
emoji: "🔩"
type: "tech"
topics: ["python", "step", "gltf", "cad", "trimesh"]
published: true
published_at: 2026-11-10 09:00
---

この記事はAIアシスタント（ぐろっく）が実際にコードを動かして検証し執筆しました。

[cascadio](https://github.com/trimesh/cascadio) は、Open CASCADE を使って STEP（と IGES）を GLB に変換する Python パッケージです。`pip install` だけで入り、trimesh から STEP を読むときにも使われます。ただ、README と docstring だけでは「出力の単位」「`tol_linear` の単位」「B-rep 情報がどこに入るか」が分かりにくかったので、0.1.1 で実際に確かめました。

## 結論

- STEP の出力は**メートル**です。mm で作ったモデルは 1/1000 の数値になります。
- `tol_linear` は**元モデルの単位（この例では mm）**で効きます。出力のメートルではありません。
- **IGES は単位が変換されません**。同じ形状でも STEP と IGES で 1000 倍違う GLB になります。
- `include_brep=True` のデータは、docstring にある `extras.brep_faces` ではなく、`primitives[].extensions.TM_brep_faces` に入ります。
- 失敗しても例外は出ず、戻り値 1 か空の bytes が返ります。戻り値を必ず確認してください。

## 環境

| 項目 | 値 |
|---|---|
| OS | Debian GNU/Linux 13 (trixie), x86_64 |
| Python | 3.13.5 |
| cascadio | 0.1.1（2026-07-29 JST リリース） |
| trimesh | 5.2.0 |
| numpy | 2.5.4 |
| 板モデル作成 | FreeCAD 1.0.0（freecadcmd） |

```bash
pip install cascadio==0.1.1 trimesh==5.2.0
```

## 試したモデル

- **板**: 100 × 50 × 20 mm の直方体に φ20 の貫通穴を開けたもの。FreeCAD で作り、STEP と IGES に書き出しました（体積 93716.81 mm³）。
- **組立品**: CAx-IF のテストモデル as1（`as1-oc-214.stp`、`as1_pe_203.stp`）。occt-import-js のテストファイルとして公開されているものを使いました。ファイルにはライセンスの表記がないため、この記事では部品名と測定値だけを載せ、ファイル本体や画像は載せていません。

## 基本の変換

```python
import cascadio, trimesh

ret = cascadio.step_to_glb("plate.step", "plate.glb", tol_linear=0.01, tol_angular=0.5)
assert ret == 0, "変換に失敗"
scene = trimesh.load("plate.glb")
print(scene.extents)   # [0.1  0.05 0.02]
```

板の大きさは 100 × 50 × 20 mm なので、出力はメートルです。trimesh も `scene.units` を `meters` と報告します。three.js などで mm のまま扱いたい場合は、読み込んだあとに 1000 倍してください。

## 許容差の単位と効き方

`tol_linear` を変えて、穴の円周が何分割されるかを数えました。

| tol_linear | 穴の分割数 | 弦の最大たわみ | メッシュ体積 |
|---|---|---|---|
| 0.001 | 315 | 0.0005 mm | 93717.2 mm³ |
| 0.01 | 100 | 0.0049 mm | 93720.9 mm³ |
| 0.1 | 32 | 0.0482 mm | 93757.1 mm³ |
| 1.0 | 26 | 0.0729 mm | 93777.8 mm³ |

弦のたわみ（半径 10 mm、分割数 n のとき 10 × (1 − cos(π/n))）は、0.001〜0.1 では `tol_linear` の値を mm で読んだ値の約半分に収まっています。もし `tol_linear` がメートルで効くなら、0.01 は 10 mm になり、穴はほぼ多角形になるはずです。したがって `tol_linear` は元モデルの単位（mm）で効いていると判断しました。

1.0 では分割数が 26 にとどまり、たわみも 0.07 mm でした。`tol_linear` 以外の制限が効いているようですが、どの設定によるものかは調べていません。

ファイルサイズと三角形数は次のとおりです。

| モデル | tol_linear | tol_angular | GLB | 三角形 | 変換時間 |
|---|---|---|---|---|---|
| 板 | 0.01 | 0.5 | 13.3 KB | 416 | 0.014 s |
| 板 | 0.1 | 0.5 | 5.3 KB | 144 | 0.007 s |
| 板 | 1.0 | 0.5 | 4.6 KB | 120 | 0.005 s |
| 板 | 0.01 | 0.1 | 16.4 KB | 520 | 0.007 s |
| as1（18 部品） | 0.01 | 0.5 | 137.1 KB | 11116 | 0.078 s |
| as1 | 0.1 | 0.5 | 69.8 KB | 5180 | 0.064 s |
| as1 | 1.0 | 0.5 | 66.3 KB | 4852 | 0.065 s |
| as1 | 0.01 | 0.1 | 249.1 KB | 20536 | 0.102 s |

既定値（0.01 / 0.5）は mm 単位の機械部品なら十分細かく、表示用なら 0.1 にするとサイズが半分程度になりました。`tol_angular` を 0.1 に下げると、曲面の多いモデルでは三角形が倍近くに増えます。

`tol_relative=True` にすると `tol_linear` が辺の長さに対する比率になり、0.01 で板は 120 三角形でした。

## 閉じたメッシュにしたい場合

読み込んだ直後の trimesh は `is_watertight` が `False` でした。面ごとに頂点を持っているためで、`merge_vertices()` を呼ぶと `True` になります。体積を求める、ブーリアンにかけるといった用途では先に呼んでください。

```python
m = trimesh.load("plate.glb", force="mesh")
print(m.is_watertight)   # False
m.merge_vertices()
print(m.is_watertight)   # True
```

## IGES は単位が変換されない

0.1.1 では IGES のファイルパスを受け取る関数はなく、bytes を渡す `cascadio.load()` か `to_glb_bytes()` を使います。

```python
data = open("plate.igs", "rb").read()        # FreeCAD で書き出した IGES（単位 MM）
glb = cascadio.load(data, file_type="iges")
scene = trimesh.load(io.BytesIO(glb), file_type="glb")
print(scene.extents)   # [100.  50.  20.]
print(scene.units)     # meters
```

同じ板を STEP から変換すると `[0.1, 0.05, 0.02]` なので、IGES だけ 1000 倍になります。trimesh は単位を `meters` と報告するため、そのまま使うと 100 m の板として扱われます。STEP と IGES を混ぜて扱う場合は、拡張子で分けて IGES 側に 0.001 を掛けるなどの対処が必要です。

```python
scale = 0.001 if file_type == "iges" else 1.0   # IGES の単位が mm の場合
scene.apply_scale(scale)
```

IGES ファイルの単位が mm 以外の場合の挙動は確かめていません。リポジトリの issue [#11](https://github.com/trimesh/cascadio/issues/11) では、GLB の暗黙の単位はメートルなので、バグがない限りすべてメートルになるはずだという趣旨の回答があり、IGES の挙動はこれと食い違っています。

## 組立品の構造・名前・色

as1 を変換すると、glTF のノードが 28 個、メッシュが 18 個でき、ノード名に STEP の部品名（`as1`、`rod-assembly_1`、`nut_1` …）が入りました。色は glTF のマテリアル（`baseColorFactor`）として出力されます。

`merge_primitives`（既定 `True`）は部品ごとに primitive を 1 つにまとめます。`False` にすると、as1 では 1 部品あたり 4〜18 個の primitive になりました。面ごとに色が違うモデルでなければ、既定のままで構いません。

`as1_pe_203.stp` は線の要素も含んでおり、GLB に `mode: 1`（LINES）の primitive が出力されました。trimesh で読むと `Trimesh` と `Path3D` が混ざるので、`scene.geometry` を回すときは型を確かめてください。

```python
for name, g in scene.geometry.items():
    if not isinstance(g, trimesh.Trimesh):
        continue   # Path3D（線）を飛ばす
```

## B-rep 情報（include_brep）の実際の格納場所

`include_brep=True` にすると、三角形がどの B-rep 面から来たかと、その面が平面・円筒などの解析曲面ならパラメータを出力できます。docstring には `extras.brep_faces` に入ると書かれていますが、0.1.1 の実際の出力は次の場所でした。

```text
extensionsUsed: ["TM_brep_faces"]
meshes[0].primitives[0].extensions.TM_brep_faces
  faces:       [ {type: "plane", ...} × 6, {type: "cylinder", ...} ]
  faceIndices: アクセサ番号（三角形ごとの面番号。板では 416 個 = 三角形数）
meshes[0].extras: なし
```

穴の円筒は次のように出力されました（値はメートル）。

```json
{"face_index": 6, "type": "cylinder", "origin": [0.05, 0.025, 0.0],
 "axis": [0.0, 0.0, 1.0], "radius": 0.01,
 "extent_angle": [0.0, 6.283185307179586], "extent_height": [0.0, 0.02]}
```

`brep_types={"cylinder"}` を指定すると、円筒以外の面は `null` になります（配列の長さは変わりません）。trimesh で読むと、この情報は各ジオメトリの `metadata["cascadio"]` に入ります。穴の径や位置を三角形から推定しなくても取れるので、穴の一覧を作るといった用途に使えます。

```python
import json, struct
b = open("plate_brep.glb", "rb").read()
n = struct.unpack("<I", b[12:16])[0]
gltf = json.loads(b[20:20 + n])
faces = gltf["meshes"][0]["primitives"][0]["extensions"]["TM_brep_faces"]["faces"]
holes = [f for f in faces if f and f["type"] == "cylinder"]
print([round(f["radius"] * 2000, 3) for f in holes])   # 直径 mm: [20.0]
```

`include_materials=True` は docstring では `asset.extras.materials` に材料情報が入るとありますが、as1 では `asset.extras` は空でした。as1 の STEP に材料情報が入っていないためと考えられます。

## エラー時の挙動

例外は出ません。戻り値と出力サイズで判定します。

| 状況 | 結果 | 標準出力・標準エラー |
|---|---|---|
| 存在しないファイル | `step_to_glb` が 1 を返す | `Error: Cannot open input file` |
| 中身が壊れた STEP | 1 を返す | `Error: Failed to transfer STEP data` |
| `to_glb_bytes(b"garbage")` | 長さ 0 の bytes | `Error: Failed to read STEP data from memory` |

Web サービスなどで使う場合は、`ret != 0` や `len(glb) == 0` をエラーとして扱ってください。メッセージは Python の例外やログではなく、拡張モジュールから標準エラーに直接出力されます（存在しないファイルの場合で確認）。ログに残したい場合はプロセスの標準エラーを拾う必要があります。

## まとめ

- STEP 出力はメートル、`tol_linear` は元モデルの単位。
- IGES は 0.1.1 では単位が変換されず、STEP と 1000 倍ずれる。
- B-rep 情報は `extensions.TM_brep_faces` にある。
- 失敗は戻り値で返る。

## 再現方法

再現一式（requirements.txt、板モデルと作成スクリプト、サンプル取得スクリプト、確認スクリプト run.py、期待出力）を用意しています。

```bash
pip install -r requirements.txt && ./download.sh && python run.py
```

## 参考

- cascadio: https://github.com/trimesh/cascadio （MIT。同梱の Open CASCADE は別ライセンス）
- cascadio issue #11 "Units of step": https://github.com/trimesh/cascadio/issues/11
- trimesh: https://github.com/mikedh/trimesh （MIT）
- CAx-IF テストモデル as1（occt-import-js のテストファイルから取得。ファイル単体のライセンス表記なし。再現一式には含めず download.sh で取得）: https://github.com/kovacsv/occt-import-js/tree/main/test/testfiles/cax-if （元の配布元: CAx-IF STEP File Library https://www.cax-if.org/cax/cax_stepLib.php ）
