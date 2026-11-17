# Bob the bot builder 🔨👷‍♂️

Bob builds RLBots for the botpack. It reads a `bob.toml` from your bot's repository, builds the bot for Windows and Linux in Docker, and writes a folder with the binaries and config files that RLBot needs.

## Runtime dependencies

- [Docker](https://www.docker.com/get-started/). Bob uses Linux containers.

## Build a bot

1. Put a `bob.toml` at the root of your bot's repository.
2. Make sure Docker is running.
3. Run `bob build bob.toml`.

The result is `bob_build/<project_name>/`. It holds the built binaries, your bot config with rewritten run commands, and the logo and loadout files.

## Configure the build

`bob.toml` has one `[[config]]` section per build. A Python bot needs four fields:

```toml
[[config]]
project_name = "MyBot"
bot_configs = ["src/bot.toml"]

[config.builder_config]
builder_type = "pyinstaller"
entry_file = "bot.spec"
```

`project_name` sets the folder name in `bob_build/`. `bot_configs` lists the bot configs to package. `builder_type` chooses the build steps, and `entry_file` is the PyInstaller spec for that builder.

See [Configuring bob.toml](docs/configuration.md) for every field, the other builders, and what Bob does to your bot config.

Find example repositories for each language here: https://github.com/swz-git/bob-example
