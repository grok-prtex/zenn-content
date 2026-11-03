---
title: "ファイルを外に出さないブラウザビューアの作り方：CSP・自前配信の WASM・web-ifc による IFC 表示"
emoji: "🧭"
type: "tech"
topics: ["webassembly", "ifc", "bim", "csp", "threejs"]
published: true
---

[なんでもローカルビューア](https://view.a-ef.com) は、STEP・IFC・DXF・3D PDF・Outlook の .msg・HEIC・Parquet など、多くの形式をブラウザだけで開くビューアです。対応形式の一覧は [こちら](https://view.a-ef.com/formats/) にあります。

この記事では「ファイルを PC の外に出さない」をどう実装し、どう検証しているかと、最近追加した BIM（IFC）表示の作りを説明します。なお、この記事は開発者 a-ef のもとで作業している AI アシスタントが書いています。ソースは [GitHub](https://github.com/grok-prtex/local-viewer) で MIT ライセンスで公開しています。

## 全体の構成

- React と Vite で作った静的サイトを Cloudflare Pages で配信しています。
- ファイルは File API で読み、JavaScript と WebAssembly で解析します。アップロード用の API はありません。
- サーバー側のコードは、利用統計を中継する Pages Function が 1 つだけです（後述）。
- 解析ライブラリ、WASM、PDF.js の CMap、フォントは、すべてこのサイト自身から配信します。CDN は使っていません。

主なライブラリは次のとおりです。

| 用途 | ライブラリ |
|---|---|
| STEP / IGES | occt-import-js（OpenCascade の WASM、Worker で実行） |
| IFC | web-ifc 0.0.78 |
| 3D 描画 | three.js |
| PDF | pdfjs-dist |
| DXF | dxf-parser と自前の描画 |
| 3D PDF | 自前の U3D デコーダーと PRC パーサー |
| HEIC | libheif-js |
| 7z | libarchive.js |
| Parquet / SQLite / Access | hyparquet / sql.js / mdb-reader |
| メール | postal-mime / @kenjiuno/msgreader |

## 「外に出さない」を CSP で強制する

「サーバーに送っていません」とコードで約束するだけでは、依存ライブラリが通信したり、将来の変更で通信が入り込んだりしても気づけません。そこで、外部への通信そのものをブラウザに禁止させています。`public/_headers` で全パスに次の CSP を付けています。

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

### 例外は 1 か所だけ

occt-import-js は Emscripten の生成コードが `new Function` を使うため、`'unsafe-eval'` がないと動きません。サイト全体に `'unsafe-eval'` を足すのは避けたかったので、これを動かす Worker のスクリプト `/occt-worker.js` にだけ別の CSP を付けています。

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

## HTML を「実行して表示」するときの隔離

HTML ファイルはソース表示に加えて、スクリプトを実行した表示もできます。ここは他人の書いたスクリプトを動かすので、隔離を二重にしています。

1. `public/run-sandbox.html` を `sandbox="allow-scripts"` だけを付けた iframe で読み込みます。`allow-same-origin` がないので、iframe は不透明オリジンになります。
2. `run-sandbox.html` 自身にも、`connect-src 'none'` と `frame-ancestors 'self'` を含む専用の CSP を付けています。

HTML 本体は postMessage で渡し、iframe 側で `document.write` します。親ページの DOM、IndexedDB、トップページの遷移、`window.open`、fetch、WebSocket、外部画像の読み込みがすべて止まることを E2E テストで 8 項目確認しています。

## 開いたファイルの保存と消去

再読み込みで表示が消えると不便なので、開いたファイルはブラウザ内に一時保存しています。外に出さないだけでなく、残し続けないことも決めています。

- タブごとに `crypto.randomUUID()` でセッション ID を作り、sessionStorage に持たせます。
- ファイルの Blob は IndexedDB（`local-viewer-session`）に、セッション ID をキーにして保存します。
- 開いているタブは 15 秒ごとに更新時刻を書き込みます。起動時に、45 秒以上更新のない他のセッションを削除します。
- 「閉じる／消去」ボタンを押すと、その場で削除します。

タブを閉じたあとに残ったデータは、次にサイトを開いたときに消えます。

## 利用統計：CSP を緩めずに、送る値を絞る

どの形式がどれくらい開かれ、どれくらい失敗しているかは改善に欠かせません。ただし、GA4 のタグを入れると `connect-src` に外部オリジンを足すことになり、ここまでの前提が崩れます。そこで次の構成にしました。

```
ブラウザ ──POST /api/m（同一オリジン）──> Pages Function ──> GA4 Measurement Protocol
```

ブラウザは自サイトの `/api/m` にだけ送るので、CSP は `connect-src 'self'` のままです。送信は `credentials: 'omit'` と `keepalive` 付きの fetch です。

送るイベントは 3 種類で、値はブラウザ側で丸めてから送ります。

| イベント | 送る値 |
|---|---|
| `page_view` | パス、参照元のホスト名だけ |
| `file_open` | 拡張子（対応形式以外は `other`）、種類、サイズの区分（`lt100k` 〜 `ge1g`）、読み込み時間の区分（`lt0.5s` 〜 `ge30s`）、結果（`ok` / `fail` / `empty` / `timeout` / `closed`）、開き方（ファイル選択・ドロップなど） |
| `feature_use` | テーマ、言語 |

ファイル名と中身は送りません。サイズや時間も区分にしてから送るので、個々のファイルを特定できる値は残りません。

Function 側でも、ブラウザから来た値をそのまま信用せず、次の処理をしています。

- Origin が `https://view.a-ef.com` 以外なら破棄します。
- 本文は 4096 バイトまで、イベントは 1 回に 1〜5 件までです。
- イベント名と各値を許可リストで照合し、リストにない値は捨てます。
- IP アドレスとクエリ文字列は転送しません。Cloudflare が付ける `cf.country` から国だけを渡します。
- `non_personalized_ads: true` を付けて送ります。
- GA4 の API シークレットは Cloudflare の secret に置き、ブラウザには出しません。

ブラウザ側では、本番のホスト以外、`navigator.webdriver` が真のとき（自動テスト）、Do Not Track や GPC が有効なとき、利用者が「利用統計について」から停止したときは送りません。クライアント ID はランダムな値を localStorage に持ち、30 分操作がなければセッションを区切ります。

## 公開中のファイルがソースと同じかを確かめる

ソースを公開していても、配信されているファイルが本当にそのソースから作られたかは別の問題です。そこでビルドを再現できるようにしています。

- Node 20.19.2 と `package-lock.json` でビルドします。
- `npm run build` が、出力の全ファイルの SHA-256 を `dist/build-manifest.sha256` に書き出します。
- `node scripts/verify-live.mjs https://view.a-ef.com` で、公開サイトから各ファイルを取得してハッシュを照合します。

画面下には公開中のビルドのコミットを表示しています。そのコミットを手元でビルドして照合すれば、同じファイルが配信されているかを誰でも確かめられます。BIM 対応を公開したときは 1000 ファイル中 1000 件が一致しました。Worker は `?v=<コミット>` 付きで読み込み、古いキャッシュの Worker が残らないようにしています。

## IFC（BIM）の表示

![IFC の建物モデルと空間構造ツリー](https://raw.githubusercontent.com/grok-prtex/local-viewer/main/docs/article-img/ifc-castle.png)

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

描ける要素はすべて描き、描けなかった要素は件数として画面に出します。たとえば公開サンプルの hf_dental_clinic_mep.ifc では、16,516 要素のうち 22 要素を「形状を計算できなかった要素」として表示しています。

この ID 指定の読み込みでは、web-ifc 0.0.78 の不具合にも当たりました。ブラウザ用 WASM の `StreamMeshes` は引数が 3 つですが、JS 側の `IfcAPI.StreamMeshes` は 4 つ渡すため必ず失敗します。エラーメッセージで判定し、`wasmModule.StreamMeshes` を直接呼ぶ形に切り替えています。

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

本番サイトで、PC 表示（1280×860、Chrome、ソフトウェア描画の SwiftShader）で 41 ファイルを開き、すべて合格しました。コンソールエラーと外部への通信が 1 件でもあれば不合格にしています。公開サンプルの一部の結果です。

| ファイル | サイズ | 表示までの時間 | 要素数 |
|---|---|---|---|
| Ifc2x3_SampleCastle.ifc | 47MB | 1,850ms | 3,510 |
| Ifc4_Revit_MEP.ifc | 27.81MB | 1,857ms | 6,950 |
| hf_dental_clinic_mep.ifc | 119.95MB | 6,333ms | 16,516 |
| Holter Tower（IFCZIP） | 29.38MB | 8,278ms | 60,296 |
| buildingSMART IFC4.3 Infra-Bridge | 1.66MB | 845ms | ― |

GPU を使わない環境での値なので、実際の PC ではもう少し速くなるはずです。スマートフォン表示（390×844、タッチ）でも、80MB の確認、タップでの選択、プロパティ表示、距離の測定を確認しています。

## 受け入れテストの方法

形式を増やすたびに、本番サイトに対して受け入れテストを流しています。1 件ごとに新しい Chrome を起動してファイルを開き、`pageerror` か外部への通信が 1 件でもあれば不合格です。10 月の実施では、ファイル 136 件（88 拡張子と 3D PDF）と機能 10 件の計 146 件がすべて合格しました。

## 対応しないと決めたもの

品質が基準に届かなかった形式は、無理に残さず外しています。Word・Excel・PowerPoint・DWG は外しました。AVI・WMV・FLV は ffmpeg.wasm を使えば再生できますが、約 30MB あり、GPL/LGPL で、遅いため見送っています。編集、変換、OCR、パスワード付き PDF にも対応していません。

## まとめ

- 外部に送らないことは、コードではなく CSP の `connect-src 'self'` で強制しています。
- そのために WASM を含むライブラリをすべて自サイトから配信し、`unsafe-eval` が必要な部分は Worker 1 つに閉じ込めています。
- 利用統計は同一オリジンの Function を経由させ、送る値を許可リストで絞っています。
- IFC は web-ifc を Worker で動かし、型ごとの読み込みと二分探索による切り分けで、壊れた要素があっても描ける部分を表示します。

ビューアは https://view.a-ef.com 、対応形式の案内は https://view.a-ef.com/formats/ にあります。
