# wordle (wordle)

Wordle

## Install the dependencies

```bash
npm install
```

### Start the app in development mode (hot-code reloading, error reporting, etc.)

```bash
quasar dev
```

### Lint the files

```bash
npm run lint
```

### Format the files

```bash
npm run format
```

### Build the app for production

```bash
quasar build
```

### Build the electron app for production

```bash
quasar build -m electron
```

### Install the electron app

```bash
mkdir -p ~/.local/bin
ln -s $PWD/dist/electron/Packaged/wordle-linux-x64/wordle ~/.local/bin
ln -s $PWD/dist/electron/UnPackaged/favicon.png ~/.local/share/icons/wordle.png
```

### Customize the configuration

See [Configuring quasar.config.js](https://v2.quasar.dev/quasar-cli-vite/quasar-config-js).
