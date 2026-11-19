# Configuring bob.toml

`bob.toml` tells Bob how to build your bot. Put it at the root of your bot's repository and keep that name. Bob reads it when you run `bob build bob.toml`.

A Python bot needs one build section:

```toml
[[config]]
project_name = "MyBot"
bot_configs = ["src/bot.toml"]

[config.builder_config]
builder_type = "pyinstaller"
entry_file = "bot.spec"
```

`[[config]]` defines one build. A repository can define several, and each build gets its own folder in `bob_build/`.

---

## Fields

`project_name` names the output folder. The config above produces `bob_build/MyBot/`. Use a name that is unique inside the botpack.

`bot_configs` lists the `bot.toml` or `script.toml` files that belong to this build. Paths are relative to the `bob.toml` file. Bob rejects paths that leave the project directory. Bob copies each config into the output folder, so the files must live in your repository.

`builder_config` selects the builder. The `builder_type` key decides which other keys Bob reads. Unknown keys are ignored, so a misspelled key leaves the build misconfigured instead of failing loudly.

---

## Builders

### pyinstaller

Use this for Python bots. Generate the spec with PyInstaller and commit it:

```bash
pyinstaller --onefile src/bot.py --paths src
```

This creates `bot.spec`. Point `entry_file` at that file:

```toml
[[config]]
project_name = "MyBot"
bot_configs = ["src/bot.toml"]

[config.builder_config]
builder_type = "pyinstaller"
entry_file = "bot.spec"
```

Bob's Python builder installs dependencies from `requirements.txt`, then runs PyInstaller once under Wine for Windows and once natively for Linux. The spec must work with both. Keep data paths in the spec in a form PyInstaller can resolve on both platforms.

The [python-example](https://github.com/RLBot/python-example) repository shows the basic setup. [Elementv5](https://github.com/VirxEC/Elementv5) shows a PyTorch bot that loads the pack's shared torch install.

### rust

Use this for Rust bots. You do not need a Dockerfile.

```toml
[[config]]
project_name = "MyRustBot"
bot_configs = ["./bot.toml"]

[config.builder_config]
builder_type = "rust"
targets = ["x86_64-pc-windows-gnu", "x86_64-unknown-linux-musl"]
bin_name = "my_bot"
```

`targets` lists the Rust targets to build. `bin_name` must match the binary name in `Cargo.toml`.

`include` lists files and folders to copy into the output. Use a plain string to keep the source name:

```toml
[config.builder_config]
builder_type = "rust"
targets = ["x86_64-pc-windows-gnu", "x86_64-unknown-linux-musl"]
bin_name = "my_bot"
include = [
  "checkpoints",
  { src = "other_thing" },
  { src = "config/extra.toml", dest = "extra.toml" },
]
```

Each entry copies one path into the output folder. A string, or a table with only `src`, keeps the source name. Set `dest` to rename the file or folder. `dest` is relative to the output folder. Paths are relative to `bob.toml`. Bob rejects sources outside the project and destinations that escape the output folder. Changing an included file triggers a new build.

### csharp

Use this for C# bots. You do not need a Dockerfile.

```toml
[[config]]
project_name = "MyCSharpBot"
bot_configs = ["example.bot.toml"]

[config.builder_config]
builder_type = "csharp"
```

Set `base_dir` when the `.csproj` file is not at the repository root:

```toml
[config.builder_config]
builder_type = "csharp"
base_dir = "Bot"
```

See the [C# example](https://github.com/VirxEC/csharp-example) for a complete bot.

### custom

Use this when the other builders cannot build your bot. It is common for C++ projects and bots with unusual toolchains.

```toml
[[config]]
project_name = "MyBot"
bot_configs = ["rlbot/bot.toml"]

[config.builder_config]
builder_type = "custom"
dockerfile = "cpp.Dockerfile"
values = { profile = "megabyte" }
```

`dockerfile` is a path relative to the `bob.toml` file. Bob renders it as a template and passes `values` to that template. The two braces in `{ profile }` are the placeholder syntax. A value from `values` replaces each placeholder before the Docker build starts.

The Dockerfile must write a tar archive to stdout. The archive needs one directory per platform, with the binaries and data for that platform inside:

```
x86_64-unknown-linux-gnu/
x86_64-pc-windows-msvc/
```

Bob's built-in Dockerfiles show this layout in [bob-cli/dockerfiles](https://github.com/RLBot/bob/tree/master/bob-cli/dockerfiles). The Docker build context is the directory that holds `bob.toml`, so paths in the Dockerfile resolve from there.

The [GGLBot template](https://github.com/SubparN0va/GGLBot/tree/bob) is a working C++ example. The `prebuilt_example` folder in [bob-example](https://github.com/swz-git/bob-example) shows the Dockerfile contract, but it copies ready-made binaries. Do not use that pattern in the botpack, where every build must compile from source.

---

## What Bob changes in your bot config

Bob rewrites the run commands after the build. It scans the built files for a Windows executable and a Linux executable, then writes their paths into `run_command` and `run_command_linux` in each output bot config.

The `run_command` value in your source config only matters when you run the bot from the repository. It does not ship. Set it to something that works locally.

Bob also copies the logo and loadout into the output folder. It reads `logo_file` and `loadout_file` from the source config and copies the files it finds next to that config. Omit `logo_file` when you have no logo. Bob then looks for `logo.png`. An empty `logo_file = ""` does not work; it points at the config's directory instead of a file.

---

## Several builds in one repository

One repository can ship several bots. Give each build its own `[[config]]` section and its own `project_name`:

```toml
[[config]]
project_name = "Necto"
bot_configs = ["necto/bot.toml"]

[config.builder_config]
builder_type = "pyinstaller"
entry_file = "necto.spec"

[[config]]
project_name = "Nexto"
bot_configs = ["nexto/bot.toml"]

[config.builder_config]
builder_type = "pyinstaller"
entry_file = "nexto.spec"
```

Bob builds each section separately. The two builds above produce `bob_build/Necto/` and `bob_build/Nexto/`.

The botpack itself uses a root `bob.toml` that lists the configs in all bot repositories:

```toml
dependencies = ["./Botpack/**/bob.toml"]
```

`dependencies` accepts glob patterns. Each match is read as another `bob.toml`.

---

## Check the result

Open `bob_build/<project_name>/` after the build. The folder contains the platform folders from your Dockerfile or builder, the rewritten bot config, and the logo and loadout files. Both `x86_64-unknown-linux-gnu/` and `x86_64-pc-windows-msvc/` should be present when your bot supports both platforms.
