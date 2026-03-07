# Installation

## From crates.io

```sh
cargo install diesel-guard
```

## Prebuilt Binaries

### macOS and Linux

```sh
curl --proto '=https' --tlsv1.2 -LsSf https://github.com/ayarotsky/diesel-guard/releases/latest/download/diesel-guard-installer.sh | sh
```

### Windows (PowerShell)

```powershell
powershell -ExecutionPolicy Bypass -c "irm https://github.com/ayarotsky/diesel-guard/releases/latest/download/diesel-guard-installer.ps1 | iex"
```

### Homebrew

```sh
brew install ayarotsky/tap/diesel-guard
```

## Verify Installation

```sh
diesel-guard --version
```

## Initialize Configuration

Generate a documented configuration file in your project root:

```sh
diesel-guard init
```

This creates a `diesel-guard.toml` with all available options and their descriptions. See [Configuration](configuration.md) for full details.
