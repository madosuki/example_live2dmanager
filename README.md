# example_live2dmanager

This is a sample implementation of
[madosuki/live2dManager](https://github.com/madosuki/live2dManager)
built with Vue 3 SFC + Vite + TypeScript.


## Required Files

Live2D Cubism Core is proprietary, so it is not included in this repository.
Download Cubism SDK for Web from the official Live2D website, then place the
Core file at the following path:

```text
public/live2dcubismcore.min.js
```

Live2D models to display are also not included in this repository. For example,
to run with the default settings, place the model files as follows:

```text
public/live2d_assets/Mao/Mao.model3.json
public/live2d_assets/Mao/...
```

If the model name or directory is different, you can change it from the input
fields on the right side of the screen.

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

- Motions and expressions are retrieved from `Live2dModel` after the model is
  loaded, then rendered as buttons.
