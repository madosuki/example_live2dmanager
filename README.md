# example_live2dmanager

Vue 3 SFC + Vite + TypeScriptで実装した
[madosuki/live2dManager](https://github.com/madosuki/live2dManager) のサンプルです。

`live2dManager`のREADMEにあるコードは古い可能性があるため、このサンプルでは
インストール済みパッケージの型定義と実装に合わせて、次のAPIを使っています。

- `new Live2dViewer(canvas, width, height)`
- `new Live2dManager(viewer)`
- `manager.initialize()`
- `new Live2dModel(modelHomeDir, modelJsonFileName, viewer, readFileFunction)`
- `model.loadAssets(preloadMotions)`
- `viewer.addModel(key, model)`
- `viewer.setCurrentModelk(key)`

## Required Files

Live2D Cubism Coreはプロプライエタリのため、このリポジトリには含めていません。
Live2D公式サイトからCubism SDK for Webを取得し、Coreファイルを次の場所に置いてください。

```text
public/live2dcubismcore.min.js
```

表示するLive2Dモデルも同じくこのリポジトリには含めていません。たとえばデフォルト設定で
起動する場合は、次のように配置します。

```text
public/models/Haru/Haru.model3.json
public/models/Haru/...
```

モデル名やディレクトリが異なる場合は、画面右側の入力欄で変更できます。

## Development

```sh
pnpm install
pnpm dev
```

## Build

```sh
pnpm build
```

## Notes

- `Live2dViewer.runSingleModel()`は内部で`requestAnimationFrame`を開始しますが停止口がありません。
  Vueのアンマウント時に確実に止めるため、このサンプルではSFC内で描画ループを管理しています。
- `Live2dViewer`の現在モデル設定メソッド名は、現行型定義では`setCurrentModelk`です。
- モーションと表情は、モデル読み込み後に`Live2dModel`から一覧を取得してボタン化しています。
