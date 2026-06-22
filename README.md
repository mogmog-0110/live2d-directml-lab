# live2d-directml-lab

D3D12 上の Live2D 描画結果に、**DirectML 推論をパイプライン内（CPU 往復なし）で適用する後処理**の実装。シャープンと単眼深度ベースのリライティングを実装。移植本体は別リポジトリ [live2d-cubism-d3d12](https://github.com/mogmog-0110/live2d-cubism-d3d12)。

## 構成

- **raw DirectML（zero readback）** — `backbuffer → compute(pack NCHW) → DML 演算 → compute(unpack) → 合成`。同一コマンドリスト上、CPU 往復なし。`DX12NeuralFx.hpp`
- **ONNX Runtime + DirectML EP** — 単眼深度（Depth-Anything-V2-Small）を**非同期ロード**し、フレーム境界で推論 → 深度テクスチャへ反映。`NeuralDepth.hpp` / `DX12NeuralRelight.hpp`

## 実装した後処理

- **シャープン** — 輝度成分のみのアンシャープ（色は変更しない）。
- **リライティング** — 単眼深度から法線を推定し可動光源でライティング。元のピクセルは変更せず光を加算する（強度 0 で入力に一致）。

<p align="center"><img src="docs/images/depth-character-form.png" width="300"></p>

深度の挙動：全画面入力では汎用の単眼深度は背景の奥行きを推定しフラットなキャラは平面として出るが、キャラ領域にクロップすると人物の概形が出る（上図、左=入力 / 右=推定深度）。これを法線に用いる。後処理はいずれも完成済みのピクセルを変更する。

## 技術メモ

同梱 ORT が 1.24 でも `onnxruntime.dll` を exe 隣に deploy しないと、Windows が `system32` の 1.17.1 を読み API 不一致でクラッシュ（SEH 例外で C++ try/catch では捕捉不可）。正しい DLL を exe 隣へ deploy して解消。

## ソース

- [`src/DX12NeuralFx.hpp`](src/DX12NeuralFx.hpp) — raw DirectML in-pipeline（zero readback）
- [`src/NeuralDepth.hpp`](src/NeuralDepth.hpp) — ORT + DirectML EP 単眼深度（非同期ロード）
- [`src/DX12NeuralRelight.hpp`](src/DX12NeuralRelight.hpp) — 加算リライト（深度 → 法線 → 可動光源）

## クレジット / ライセンス

コードは MIT。**Live2D Cubism SDK**（© Live2D Inc.）使用、各 [SDK 使用許諾](https://www.live2d.com/eula/live2d-sdk-license-agreement_jp.html) に従う。掲載モデルは [Free Material License](https://www.live2d.com/eula/live2d-free-material-license-agreement_jp.html) のサンプル。深度モデル Depth-Anything-V2-Small（Apache-2.0）。**SDK・モデル・深度モデルは非同梱。**
