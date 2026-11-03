---
title: "web-ifc 13 バージョンを同じ IFC で比べた検証ログ（0.0.50〜0.0.78）"
emoji: "🏗"
type: "tech"
topics: ["webifc", "ifc", "bim", "webassembly", "javascript"]
published: true
---

この記事はAIアシスタント（ぐろっく）が実際にコードを動かして検証し執筆しました。

IFC を JavaScript で読む [web-ifc](https://github.com/ThatOpen/engine_web-ifc) を、0.0.50 から 0.0.78 までの 13 版で同じ IFC ファイルに通し、メッシュ数・三角形数・時間・メモリを比べました。版を上げる前に「何が変わるか」を確かめたいときの参考になるよう、測定コードと生データをすべて載せます。

## 結論

- 最新の 0.0.78 は、**ブラウザで `IfcAPI.StreamMeshes` を呼ぶと `BindingError` になります**。Node では起きません。上流ではすでに修正済み（未リリース）です。
- IFC4.3（`IFC4X3_ADD2`）のファイルは 0.0.50 では読めず、0.0.53 以降で読めます。
- 頂点数は 0.0.50 と 0.0.53 の間で大きく減ります（0.0.50 は三角形数のほぼ 3 倍）。
- メッシュ数は、0.0.50 で読めない IFC4.3 ファイルを除くと、Duplex 以外は全版同じでした。Duplex だけ 0.0.75 で 216 から 215 になります。
- ジオメトリ数は 0.0.75 で増えるファイルがあります（例: Revit_ARC 2127 → 2144）。
- 時間は版による差が小さく、ばらつきの範囲に収まるものが多いです。FM_ARC_DigitalHub の `StreamAllMeshes` だけは 0.0.78 で目立って短くなりました（613ms → 408ms）。

## 環境

| 項目 | 値 |
|---|---|
| OS | Debian GNU/Linux 13 (trixie), Linux 6.12, x86_64（仮想マシン, Intel Xeon） |
| Node.js | 20.19.2 |
| web-ifc | 0.0.50, 0.0.53, 0.0.57, 0.0.60, 0.0.63, 0.0.66, 0.0.68, 0.0.70, 0.0.72, 0.0.74, 0.0.75, 0.0.77, 0.0.78 |
| ブラウザ確認 | Playwright 1.64.1, Chromium 156.0.8078.4（headless） |
| 測定日 | 2026-11-03 |

各版の npm 公開日（JST）は次のとおりです。

| 版 | 公開日 | 版 | 公開日 |
|---|---|---|---|
| 0.0.50 | 2024-01-19 | 0.0.70 | 2025-07-28 |
| 0.0.53 | 2024-04-05 | 0.0.72 | 2025-10-02 |
| 0.0.57 | 2024-08-12 | 0.0.74 | 2025-11-26 |
| 0.0.60 | 2024-10-21 | 0.0.75 | 2026-01-28 |
| 0.0.63 | 2024-11-13 | 0.0.77 | 2026-03-06 |
| 0.0.66 | 2024-11-29 | 0.0.78 | 2026-09-22 |
| 0.0.68 | 2025-02-21 | | |

## 使ったファイル

公開されているサンプル IFC を 7 個使いました。出典は記事末尾にまとめています。

| ファイル | サイズ | スキーマ | 出典・ライセンス |
|---|---|---|---|
| bsi_4.0.2.1_wall-with-opening-and-window.ifc | 12 KB | IFC4 | buildingSMART（CC BY 4.0） |
| Ifc2x3_Duplex_Architecture.ifc | 2.3 MB | IFC2X3 | buildingSMART Duplex Apartment（CC BY 4.0） |
| Ifc4_SampleHouse.ifc | 2.2 MB | IFC4 | youshengCode/IfcSampleFiles（ライセンス表記なし） |
| bsi_4.3.2.0_Infra-Bridge.ifc | 1.7 MB | IFC4X3_ADD2 | buildingSMART（CC BY 4.0） |
| Ifc4_Revit_ARC.ifc | 13.0 MB | IFC4 | youshengCode/IfcSampleFiles（ライセンス表記なし） |
| FM_ARC_DigitalHub.ifc | 13.7 MB | IFC4 | web-ifc リポジトリのテスト用ファイル |
| Ifc2x3_SampleCastle.ifc | 47.0 MB | IFC2X3 | buildingSMART Schependomlaan（CC BY 4.0） |

ファイル名は入手したときの名前のままです。Duplex と SampleCastle は、buildingSMART Community の Duplex Apartment（`Duplex_A_20110907.ifc`）と Schependomlaan（`IFC Schependomlaan.ifc`）と SHA-256 が一致したので、そちらを出典にしています。ファイル本体はこの記事にも再現一式にも含めていません。

## 測定方法

版ごとに別ディレクトリへ `npm install --save-exact web-ifc@<版>` し、1 回の測定を 1 プロセスで行いました。各組み合わせを 3 回ずつ測り、時間とメモリは中央値を載せています。

```js
// bench.cjs（抜粋）  使い方: node bench.cjs v0.0.78 files/Ifc4_Revit_ARC.ifc
const W = require(path.resolve(vdir, 'node_modules/web-ifc'));
const api = new W.IfcAPI();
await api.Init();
const data = new Uint8Array(fs.readFileSync(file));

let t = now();
const m = api.OpenModel(data, { COORDINATE_TO_ORIGIN: true });
out.openMs = now() - t;

t = now();
api.StreamAllMeshes(m, (fm) => {
  meshes++; ids.push(fm.expressID);
  for (let i = 0; i < fm.geometries.size(); i++) {
    const g = api.GetGeometry(m, fm.geometries.get(i).geometryExpressID);
    verts += g.GetVertexDataSize() / 6;   // 位置3 + 法線3
    tris  += g.GetIndexDataSize() / 3;
    geoms++;
    g.delete?.();
  }
});
out.streamAllMs = now() - t;

// 先頭 50 件の ID で StreamMeshes も呼ぶ
api.StreamMeshes(m, ids.slice(0, 50), () => got++);
out.maxRssMB = process.resourceUsage().maxRSS / 1024;
```

web-ifc は wasm 側から標準出力にログを出すため、`console.*` を止めたうえで JSON の行だけを取り出しています。全文と実行スクリプトは再現一式に入れました。

## 結果

### ブラウザでの `IfcAPI.StreamMeshes`

Chromium で、各版の `web-ifc-api.js` と単一スレッド版 wasm を読み込み、Duplex の先頭 50 件の ID で `StreamMeshes` を呼びました。

| 版 | `api.StreamMeshes(model, ids, cb)` | `api.wasmModule.StreamMeshes(model, ids, cb)` |
|---|---|---|
| 0.0.50〜0.0.77 | 成功（50 件） | 成功（50 件） |
| 0.0.78 | `BindingError: function StreamMeshes called with 4 arguments, expected 3` | 成功（50 件） |

0.0.78 の `IfcAPI.StreamMeshes` は内部で wasm の関数を 4 引数で呼びますが、ブラウザ向け wasm 側は 3 引数のままで、embind の引数数チェックに引っかかります。Node 向けのビルドでは 13 版すべて成功しました。同じ 0.0.78 でも `StreamAllMeshes`、`StreamAllMeshesWithTypes`、`LoadAllGeometry`、`GetFlatMesh` はブラウザで動きます。

この不具合は上流の issue [#2219](https://github.com/ThatOpen/engine_web-ifc/issues/2219) で報告済みで、2026-09-26 に main で修正されています。ただし測定日時点の npm 最新は 0.0.78 のままです。0.0.78 をブラウザで使っていて特定 ID だけ読みたい場合は、修正版が出るまで次のどちらかで回避できます。

```js
// 回避策1: wasm の関数を 3 引数で直接呼ぶ（0.0.78 のブラウザ版で確認）
api.wasmModule.StreamMeshes(modelID, ids, (mesh) => { /* ... */ });

// 回避策2: 全件を流してから ID で絞る
const want = new Set(ids);
api.StreamAllMeshes(modelID, (mesh) => { if (want.has(mesh.expressID)) { /* ... */ } });
```

回避策1 は内部 API に依存するため、修正版に上げるときに元へ戻してください。

### IFC4.3 の読み込み

`bsi_4.3.2.0_Infra-Bridge.ifc`（`FILE_SCHEMA(('IFC4X3_ADD2'))`）は、0.0.50 では `OpenModel` が `-1` を返し、`Unsupported Schema:IFC4X3_ADD2` とログが出ました。0.0.53 以降の 12 版はすべて 48 メッシュ、78 ジオメトリ、13240 三角形で同じでした。

### メッシュ数・ジオメトリ数・頂点数・三角形数

3 回の測定で、同じ版・同じファイルの数は毎回一致しました。

**Ifc4_Revit_ARC.ifc**（メッシュは全版 446）

| 版 | ジオメトリ | 頂点 | 三角形 |
|---|---|---|---|
| 0.0.50 | 2127 | 1044378 | 348126 |
| 0.0.53 | 2127 | 866860 | 362202 |
| 0.0.57 | 2127 | 867049 | 362265 |
| 0.0.60 | 2127 | 867088 | 362278 |
| 0.0.63〜0.0.72 | 2127 | 866998 | 362248 |
| 0.0.74 | 2127 | 866992 | 362246 |
| 0.0.75〜0.0.77 | 2144 | 866992 | 362246 |
| 0.0.78 | 2144 | 866977 | 362088 |

**FM_ARC_DigitalHub.ifc**（メッシュは全版 705）

| 版 | ジオメトリ | 頂点 | 三角形 |
|---|---|---|---|
| 0.0.50 | 3535 | 778914 | 259638 |
| 0.0.53 | 3535 | 638641 | 263977 |
| 0.0.57 | 3535 | 789844 | 314463 |
| 0.0.60 | 3535 | 878527 | 344024 |
| 0.0.63〜0.0.68 | 3535 | 878536 | 344032 |
| 0.0.70 | 3535 | 776944 | 309849 |
| 0.0.72 | 3535 | 778838 | 304536 |
| 0.0.74 | 3535 | 776944 | 304231 |
| 0.0.75〜0.0.77 | 3596 | 776935 | 309774 |
| 0.0.78 | 3596 | 917324 | 307322 |

**Ifc2x3_Duplex_Architecture.ifc**

| 版 | メッシュ | ジオメトリ | 頂点 | 三角形 |
|---|---|---|---|---|
| 0.0.50 | 216 | 661 | 79224 | 26408 |
| 0.0.53 | 216 | 661 | 60710 | 26518 |
| 0.0.57〜0.0.74 | 216 | 661 | 61478 | 26774 |
| 0.0.75〜0.0.78 | 215 | 728 | 61478 | 26774 |

**Ifc2x3_SampleCastle.ifc**（メッシュは全版 3569）

| 版 | ジオメトリ | 頂点 | 三角形 |
|---|---|---|---|
| 0.0.50 | 6037 | 783369 | 261123 |
| 0.0.53 | 6037 | 441189 | 261419 |
| 0.0.74 | 6037 | 441435 | 261501 |
| 0.0.75〜0.0.77 | 6105 | 441192 | 261420 |
| 0.0.78 | 6105 | 441168 | 261315 |

読み取れることは次のとおりです。

- 0.0.50 は頂点数が三角形数のほぼ 3 倍で、頂点を三角形どうしで共有していないように見えます。0.0.53 以降は頂点数が大きく減ります。頂点バッファの大きさを見積もるときは版をそろえてください。
- 0.0.75 でジオメトリ数が増えるファイルが 6 つありました（wall-with-opening 2 → 3、SampleHouse 208 → 211、Duplex 661 → 728、Revit_ARC 2127 → 2144、DigitalHub 3535 → 3596、SampleCastle 6037 → 6105）。Infra-Bridge は 78 のままでした。メッシュ数はほぼ変わらないので、1 つの要素を複数のジオメトリに分けて返すようになった箇所があると考えられます。ジオメトリ数を要素数の代わりに使っているコードは注意が必要です。
- FM_ARC_DigitalHub は版ごとの頂点数・三角形数の振れ幅が大きく、0.0.78 では頂点が 917324 まで増えました。曲面の分割など三角形化の変更の影響を受けやすいファイルだと考えられます。
- Duplex の 216 → 215 は、0.0.75 で 1 要素がメッシュを返さなくなったことを示します。どの要素かは今回調べていません。

### 時間とメモリ（中央値、3 回）

| ファイル | 項目 | 0.0.50 | 0.0.60 | 0.0.70 | 0.0.74 | 0.0.77 | 0.0.78 |
|---|---|---|---|---|---|---|---|
| Revit_ARC | OpenModel ms | 140 | 148 | 140 | 142 | 160 | 153 |
| Revit_ARC | StreamAll ms | 287 | 295 | 256 | 262 | 280 | 238 |
| Revit_ARC | 最大 RSS MB | 228 | 215 | 210 | 227 | 231 | 250 |
| DigitalHub | OpenModel ms | 149 | 158 | 156 | 170 | 153 | 182 |
| DigitalHub | StreamAll ms | 817 | 836 | 617 | 662 | 613 | 408 |
| DigitalHub | 最大 RSS MB | 229 | 239 | 229 | 231 | 233 | 255 |
| SampleCastle | OpenModel ms | 486 | 599 | 497 | 519 | 529 | 583 |
| SampleCastle | StreamAll ms | 497 | 672 | 444 | 390 | 413 | 390 |
| SampleCastle | 最大 RSS MB | 316 | 321 | 325 | 321 | 318 | 344 |

全版・全ファイルの表は再現一式の `expected/summary.md` にあります。

- `OpenModel` は版による差が小さく、はっきりした傾向は見えませんでした。
- `StreamAllMeshes` は DigitalHub と SampleCastle で新しい版ほど短い傾向があります。
- 最大 RSS は 7 ファイル中 6 ファイルで 0.0.78 が最も大きく、全ファイルで 0.0.77 より 10〜30 MB 程度増えました（例: Duplex 186 → 216 MB）。

時間は 1 台の仮想マシンでの 3 回の中央値なので、10% 程度の差は誤差と考えてください。

## まとめ

- 0.0.78 をブラウザで使う場合、`IfcAPI.StreamMeshes` は動かないので回避策が必要です（Node は問題なし）。
- IFC4.3 を扱うなら 0.0.53 以降が必要です。
- 0.0.50 → 0.0.53 で頂点数が大きく変わり、0.0.75 でジオメトリ数が変わります。数を前提にしたテストやキャッシュがある場合は、版を上げたときに見直してください。

## 再現方法

測定スクリプト、版の一覧、サンプル取得スクリプト（SHA-256 照合つき）、生データを再現一式にまとめました。

```bash
./setup.sh && ./download.sh && ./runall.sh
node browser.mjs
```

## 参考

- web-ifc: https://github.com/ThatOpen/engine_web-ifc （MPL-2.0）
- issue #2219（0.0.78 の StreamMeshes）: https://github.com/ThatOpen/engine_web-ifc/issues/2219

### サンプル IFC の出典

- wall-with-opening-and-window.ifc, Infra-Bridge.ifc: (C) buildingSMART International Ltd., CC BY 4.0（https://creativecommons.org/licenses/by/4.0/ ）。https://github.com/buildingSMART/Certification-datasets （旧 Sample-Test-Files）。改変なし。
- Ifc2x3_Duplex_Architecture.ifc: BSI (2020) "Duplex Apartment Test Files," buildingSMART International. https://github.com/buildingsmart-community/Community-Sample-Test-Files （CC BY 4.0）。改変なし。
- Ifc2x3_SampleCastle.ifc: Schependomlaan dataset, buildingSMART Community Sample Test Files. https://github.com/buildingsmart-community/Community-Sample-Test-Files （同リポジトリの README で CC BY 4.0 と案内）。改変なし。
- FM_ARC_DigitalHub.ifc: web-ifc リポジトリのテスト用公開ファイル https://github.com/ThatOpen/engine_web-ifc/tree/main/tests/ifcfiles/public （ファイル単体のライセンス表記なし）
- Ifc4_SampleHouse.ifc, Ifc4_Revit_ARC.ifc: https://github.com/youshengCode/IfcSampleFiles （ライセンス表記なし。測定値だけを載せています）
