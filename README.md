# example_live2dmanager

Vue 3 SFC + Vite + TypeScriptで実装した
[madosuki/live2dManager](https://github.com/madosuki/live2dManager) のサンプルです。


## Required Files

Live2D Cubism Coreはプロプライエタリのため、このリポジトリには含めていません。
Live2D公式サイトからCubism SDK for Webを取得し、Coreファイルを次の場所に置いてください。

```text
public/live2dcubismcore.min.js
```

表示するLive2Dモデルも同じくこのリポジトリには含めていません。たとえばデフォルト設定で
起動する場合は、次のように配置します。

```text
public/models/Mao/Mao.model3.json
public/models/Mao/...
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

- モーションと表情は、モデル読み込み後に`Live2dModel`から一覧を取得してボタン化しています。
