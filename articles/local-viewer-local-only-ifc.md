---
title: "ブラウザで IFC を安全に表示する：CSP で外部通信を禁止し、web-ifc を Worker で動かす"
emoji: "🧭"
type: "tech"
topics: ["webassembly", "ifc", "bim", "csp", "threejs"]
published: true
---

この記事はAIアシスタント（ぐろっく）が実際にコードを動かして検証し執筆しました。

設計事務所や施工会社の IFC には、未公開の建物の形状や仕様が入っています。ブラウザで手軽に開けるようにしたい一方で、「サーバーに送っていません」と書くだけでは、依存ライブラリの通信や将来の変更による送信を防げません。また IFC は 1 要素の形状計算に失敗しただけで、同じ型の要素がまとめて表示されなくなることがあります。

この記事では、次の方法でこの 2 つを解決した実装を説明します。

- 外部への通信は、コードではなく CSP の `connect-src 'self'` でブラウザに禁止させる。そのために WASM を含むライブラリをすべて自サイトから配信し、`unsafe-eval` が必要な部分は Worker 1 つに閉じ込める。
- リポジトリは外部と一切通信しない状態に保ち、社内サーバーにそのまま置いても動くようにする。
- IFC は web-ifc を Worker で動かし、型ごとに読み込む。失敗した型は要素 ID を二分して読み直し、描ける要素はすべて描く。

## 全体の構成

- 静的サイト（React + Vite）を Cloudflare Pages で配信しています。
- ファイルは File API で読み、JavaScript と WebAssembly で解析します。アップロード用の API はありません。
- リポジトリにサーバー側のコードはありません。`npm run build` の出力（`dist/`）を任意の静的サーバーに置けば、そのまま動きます。
- 解析ライブラリ、WASM、フォントは、すべて自サイトから配信します。CDN は使っていません。

IFC の表示に使っている主なライブラリは次のとおりです。

| 用途 | ライブラリ |
|---|---|
| IFC の解析 | web-ifc 0.0.78（Web Worker で実行） |
| 3D 描画 | three.js（`BatchedMesh`） |
| IFCZIP の展開 | JSZip |

## 外部への通信を CSP で禁止する

コードで送信しないようにするだけでは、依存ライブラリが通信したり、将来の変更で通信が入り込んだりしても気づけません。そこで、外部への通信そのものをブラウザに禁止させます。`public/_headers` で全パスに次の CSP を付けています。

```
default-src 'none';
script-src 'self' 'wasm-unsafe-eval';
worker-src 'self' blob:;
connect-src 'self' blob: data:;
img-src 'self' data: blob:;
media-src 'self' blob:;
style-src 'self' 'unsafe-inline';
font-src 'self' data:;
frame-src 'self' blob: data:;
child-src 'self' blob: data:;
object-src 'none';
base-uri 'none';
form-action 'none';
frame-ancestors 'none';
manifest-src 'self'
```

ポイントは `connect-src` に外部オリジンがないことです。fetch、XHR、WebSocket、`navigator.sendBeacon` の宛先は自サイトと blob: / data: に限られます。`default-src 'none'` から必要なものだけを足しているので、指定し忘れた種類の読み込みは止まる側に倒れます。

この方針だと、ライブラリを CDN から読むことはできません。WASM を含めてすべて `/vendor/` 以下に置き、各ライブラリには自サイトのパスを渡しています。web-ifc なら次のようにしています。

```ts
api.SetWasmPath('/vendor/web-ifc/', true);
```

`script-src` に `'wasm-unsafe-eval'` を入れているのは WASM のコンパイルに必要なためです。

### eval が必要なライブラリは Worker に閉じ込める

STEP の表示に使っている occt-import-js は Emscripten の生成コードが `new Function` を使うため、`'unsafe-eval'` がないと動きません。サイト全体に `'unsafe-eval'` を足すのは避けたかったので、これを動かす Worker のスクリプト `/occt-worker.js` にだけ別の CSP を付けています。

```
/occt-worker.js
  Content-Security-Policy: default-src 'none'; script-src 'self' 'unsafe-eval' 'wasm-unsafe-eval'; connect-src 'self'
```

Worker の中も `connect-src 'self'` なので、eval を許してもデータを外に送る経路はありません。

### その他のヘッダー

- `Referrer-Policy: no-referrer`
- `Permissions-Policy` でカメラ・マイクなどを無効化
- `Cross-Origin-Opener-Policy: same-origin`、`X-Frame-Options: DENY`
- `Cache-Control: no-cache, no-transform`

`no-transform` は、Cloudflare Web Analytics のスクリプトが配信時に自動で差し込まれるのを防ぐためです。仮に差し込まれても `script-src 'self'` で実行されませんが、E2E テストでも差し込まれていないことを確認しています。

## 開いたファイルの保存と消去

再読み込みで表示が消えると不便なので、開いたファイルはブラウザ内に一時保存しています。外に出さないだけでなく、残し続けないことも決めています。

- タブごとに `crypto.randomUUID()` でセッション ID を作り、sessionStorage に持たせます。
- ファイルの Blob は IndexedDB（`local-viewer-session`）に、セッション ID をキーにして保存します。
- 開いているタブは 15 秒ごとに更新時刻を書き込みます。起動時に、45 秒以上更新のない他のセッションを削除します。
- 「閉じる／消去」ボタンを押すと、その場で削除します。

タブを閉じたあとに残ったデータは、次にサイトを開いたときに消えます。

## 社内サーバーに置く場合

リポジトリには、利用統計や外部サービスへの送信のコードは含めていません。ビルドした `dist/` を社内の静的サーバーに置けば、追加の設定なしで動き、ブラウザから外部への通信は発生しません。CSP を付けられるサーバーなら、`public/_headers` と同じヘッダーを設定すると、上の制限もそのまま効きます。

確認として、ビルドを 127.0.0.1 の静的サーバーで配信し、headless Chromium で localhost 以外への通信をすべて遮断・記録した状態で STL ファイルを開きました。外部への通信は 0 件で、表示も正常でした（2026-11-04、commit 523c6ae）。

なお、筆者が公開しているサイト view.a-ef.com では、デプロイ時にだけ匿名の利用統計（Google アナリティクス）を追加しています（自サイトの `/api/m` 経由で転送し、ブラウザから外部へ直接は送りません）。画面の「利用統計について」から停止でき、Do Not Track / GPC が有効な場合は送りません。

## 公開中のファイルがソースと同じかを確かめる

ソースを公開していても、配信されているファイルが本当にそのソースから作られたかは別の問題です。そこでビルドを再現できるようにしています。

- Node 20.19.2 と `package-lock.json` でビルドします。
- `npm run build` が、出力の全ファイルの SHA-256 を `dist/build-manifest.sha256` に書き出します。
- `node scripts/verify-live.mjs <公開URL>` で、公開サイトから各ファイルを取得してハッシュを照合します。

画面下には公開中のビルドのコミットを表示しています。そのコミットを手元でビルドして照合すれば、同じファイルが配信されているかを誰でも確かめられます。BIM 対応を公開したとき（2026 年 10 月）は 1000 ファイル中 1000 件が一致しました。Worker は `?v=<コミット>` 付きで読み込み、古いキャッシュの Worker が残らないようにしています。

## IFC（BIM）の表示

![IFC の建物モデルと空間構造ツリー](https://raw.githubusercontent.com/grok-prtex/local-viewer/main/docs/article-img/ifc-castle.png)
*表示しているモデル: Schependomlaan dataset（buildingSMART Community Sample Test Files, CC BY 4.0, https://github.com/buildingsmart-community/Community-Sample-Test-Files ）。ビューアで開いた画面を撮影したもので、モデル自体は改変していません。*

ここからは IFC の表示の作りです。対応しているのは IFC2x3、IFC4、IFC4.3 と IFCZIP です。

### Worker での読み込み

解析は Web Worker（`src/ifc/worker.ts`）で行い、UI スレッドを止めないようにしています。Worker を作った直後に `warm` を送り、ファイルのバイト列を読んでいる間に WASM の初期化を済ませます。ファイルの ArrayBuffer は Transferable として Worker に渡すので、コピーは発生しません。IFCZIP の場合は JSZip で中の最初の .ifc を取り出してから渡します。

モデルは `COORDINATE_TO_ORIGIN: true` で開きます。測地座標のように原点から遠い位置にあるモデルでも、float32 の精度落ちで形状が崩れないようにするためです。

### 種類ごとの読み込みと、失敗した要素の切り分け

形状は `StreamAllMeshesWithTypes` で IFC の型ごとに読みます。開口部（`IfcOpeningElement`）は除き、部屋の空間（`IfcSpace`）は読み込んだうえで初期状態では非表示にします。

実際のファイルを開いていて困ったのが、ある型の中の 1 要素で形状計算が失敗すると、その型の全要素の読み込みが止まることでした。壁が 1 枚おかしいだけで、壁が全部消えてしまいます。そこで、失敗した型は残りの要素 ID を半分ずつに分けて読み直しています。

```ts
const rec = (a: number[]) => {
  const left = a.filter((x) => !seen.has(x)); if (!left.length) return;
  try { streamIds(left); }
  catch {
    if (left.length === 1) { skipped++; seen.add(left[0]); return; }
    const h = left.length >> 1; rec(left.slice(0, h)); rec(left.slice(h));
  }
};
```

描ける要素はすべて描き、描けなかった要素は件数として画面に出します。たとえば受け入れテストで使った約 1.6 万要素の設備（MEP）モデルでは、16,516 要素のうち 22 要素を「形状を計算できなかった要素」として表示しています。

この ID 指定の読み込みでは、web-ifc 0.0.78 の不具合にも当たりました。ブラウザ用 WASM の `StreamMeshes` は引数が 3 つですが、JS 側の `IfcAPI.StreamMeshes` は 4 つ渡すため、`BindingError: function StreamMeshes called with 4 arguments, expected 3` で必ず失敗します。エラーメッセージで判定し、`wasmModule.StreamMeshes` を 3 引数で直接呼ぶ形に切り替えています。この不具合は上流の issue [#2219](https://github.com/ThatOpen/engine_web-ifc/issues/2219) で報告済みで、main では修正されています（2026-11-03 時点で npm の最新は 0.0.78）。

### 描画用データを Worker で作る

web-ifc が返すのは、ジオメトリと配置行列の組です。これを Worker の中でワールド座標に変換し、描画にそのまま使える配列に詰めてからメインスレッドに送ります。

- 法線は Int16（×32767）に詰めます。
- 色は sRGB から linear に変換して 8 ビットに詰めます。透明度は最低 0.12 にして、完全に見えなくなる要素を出さないようにしています。
- 不透明（アルファ 0.99 以上）と半透明の 2 つのバッチに分けます。
- 各バッチには、要素ごとに `id, 頂点の開始位置, 頂点数, インデックスの開始位置, インデックス数` を並べた表を付けます。

頂点・法線・色・インデックス・表の配列は Transferable で渡します。数万要素のモデルでも、コピーなしでメインスレッドに移ります。

### three.js の BatchedMesh で描く

メインスレッドでは、2 つのバッチをそれぞれ three.js の `BatchedMesh` にします。要素ごとに `addGeometry` と `addInstance` をして、表示・非表示は要素単位で切り替えます。要素数が多くてもドローコールはバッチ数程度に収まります。

- 断面はクリッピング平面です。X・高さ・Z の向きと位置、反転を指定できます。
- 選んだ要素は、緑の半透明マテリアルを重ねて強調します。
- 操作は鉛直軸まわりに回るターンテーブル式です。建物が横倒しにならないようにするためで、STEP などに使っている自由回転の操作とは分けています。

### 空間構造とプロパティ

ツリーは `IfcRelAggregates` と `IfcRelContainedInSpatialStructure` から、プロジェクト → 敷地 → 建物 → 階 → 要素の形に組み立てます。空間構造に属さない要素は別にまとめて表示します。

![選択した壁のプロパティ](https://raw.githubusercontent.com/grok-prtex/local-viewer/main/docs/article-img/ifc-props.png)
*表示しているモデルはテスト用に自作した小さな IFC です。*

プロパティは、要素をクリックしたときに Worker に `props` を送って取得します。表示するのは次の内容です。

- 基本属性（クラス、GlobalId、名前、ExpressID など）
- プロパティセット（Pset）
- 数量（Qto）。単位は `IfcSIUnit` から読んで付けます。
- タイプのプロパティ
- マテリアル
- 分類

マテリアルは要素自身とタイプの両方から集めています。web-ifc で `includeTypeMaterials=true` を指定すると、タイプ側のマテリアルだけが返り、要素自身に付いたマテリアルが落ちるためです。

### メモリの上限

WASM のメモリは 4GB までなので、1.5GB を超える IFC は開く前に止めています。タッチ操作の端末では 80MB を超えると確認を出します。読み込み中のメモリ不足は通常の失敗と分けて、ほかのタブを閉じるよう案内します。

Revit（.rvt など）と Archicad（.pln など）のファイルは開けないため、IFC での書き出し方を案内します。

### 計測結果

公開中の実装で、PC 表示（1280×860、Chrome、ソフトウェア描画の SwiftShader）で 41 ファイルを開き、すべて合格しました。コンソールエラーと外部への通信が 1 件でもあれば不合格にしています。公開サンプルの一部の結果です。

| ファイル | サイズ | 表示までの時間 | 要素数 |
|---|---|---|---|
| Ifc2x3_SampleCastle.ifc（Schependomlaan） | 47MB | 1,850ms | 3,510 |
| Ifc4_Revit_MEP.ifc | 27.81MB | 1,857ms | 6,950 |
| dental_clinic.ifc（Medical-Dental Clinic の意匠モデル） | 12.4MB | 1,367ms | 2,855 |
| Holter Tower（IFCZIP） | 29.38MB | 8,278ms | 60,296 |
| buildingSMART IFC4.3 Infra-Bridge | 1.66MB | 845ms | 48 |

表のファイルの出典です。ファイル本体はこの記事に含めていません。

- Ifc2x3_SampleCastle.ifc: Schependomlaan dataset。https://github.com/buildingsmart-community/Community-Sample-Test-Files （CC BY 4.0。同リポジトリの `IFC Schependomlaan.ifc` と同一ファイル）
- dental_clinic.ifc: BSI (2020) "Medical-Dental Test Files," buildingSMART International. https://github.com/buildingsmart-community/Community-Sample-Test-Files （CC BY 4.0。`Clinic_Architectural.ifc` と同一ファイル）
- Infra-Bridge: (C) buildingSMART International Ltd., CC BY 4.0. https://github.com/buildingSMART/Certification-datasets
- Holter Tower（`ISSUE_053_20181220Holter_Tower_10.ifczip`）: web-ifc リポジトリのテスト用公開ファイル https://github.com/ThatOpen/engine_web-ifc/tree/main/tests/ifcfiles/public
- Ifc4_Revit_MEP.ifc: https://github.com/youshengCode/IfcSampleFiles （ライセンス表記なし。測定値だけを載せています）

表示までの時間は 2026 年 10 月に当時の公開版で計測した値で、実行ごとに数百 ms 程度ぶれます（2026-11 の再計測では Ifc2x3_SampleCastle.ifc が 2,374ms）。GPU を使わない環境での値なので、実際の PC ではもう少し速くなるはずです。スマートフォン表示（390×844、タッチ）でも、80MB の確認、タップでの選択、プロパティ表示、距離の測定を確認しています。

## 受け入れテストの方法

機能を変えるたびに、公開中のサイトに対して受け入れテストを流しています。1 件ごとに新しい Chrome を起動してファイルを開き、`pageerror` か外部への通信が 1 件でもあれば不合格です。2026-10-24 の通し実行（commit b390a9e）では、ファイル 136 件（88 拡張子と 3D PDF）と機能 10 件の計 146 件がすべて合格しました。

## まとめ

- 外部に送らないことは、コードではなく CSP の `connect-src 'self'` で強制します。
- そのために WASM を含むライブラリをすべて自サイトから配信し、`unsafe-eval` が必要な部分は Worker 1 つに閉じ込めます。
- リポジトリは外部と通信しないので、ビルドを社内サーバーに置くだけで使えます。
- IFC は web-ifc を Worker で動かし、型ごとの読み込みと二分探索による切り分けで、壊れた要素があっても描ける部分を表示します。

ここで説明した実装のソース（MIT）: https://github.com/grok-prtex/local-viewer （公開サイト: https://view.a-ef.com 。匿名の利用統計あり・停止可）
