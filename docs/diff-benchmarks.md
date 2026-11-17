# BOBDIFF v3 tests on botpack releases

We generated patches from pairs of extracted [RLBot/botpack release archives](https://github.com/RLBot/botpack/releases). We compared the generator at `44af5de` with the generator that supplies the archive length to Zstandard. Both use level 22 and the same per-file delta settings.

## Release results

Each time covers one `bob diff` run on a Ryzen 9 7940HS with Rust 1.93.1, using the Linux musl release build. Times exclude download, extraction, and patch application. Patch sizes include the v3 header.

| Release pair | Changes in the archives | Patch bytes, before → after | Seconds, before → after |
|---|---|---:|---:|
| `r-23` → `r-24` | Eight rebuilt Linux binaries | 36,166,321 → 36,166,324 | 6.249 → 5.948 |
| `r-24` → `r-25` | Adds PhoenixCS | 1,699,473 → 1,699,500 | 1.139 → 0.784 |
| `r-25` → `r-26` | Adds TensorBot | 70,501,733 → 70,501,736 | 12.565 → 12.522 |
| `r-26` → `r-27` | Replaces the TensorBot binary | 26,545,954 → 26,545,957 | 3.738 → 2.967 |
| `r-27` → `r-28` | No changes in files | 645 → 646 | 0.481 → 0.096 |
| `r-25` → `r-24` | Removes PhoenixCS | 579 → 580 | 0.378 → 0.064 |
| `r-47` → `r-48` | Changes the Dice loadout and build metadata | 3,953 → 4,107 | 1.360 → 0.977 |
| `r-48` → `r-49` | Adds ByteBotpack | 2,328,819 → 2,328,833 | 1.735 → 1.289 |
| `r-49` → `r-50` | Replaces ByteBotpack with two bots | 33,551,848 → 33,551,729 | 7.298 → 6.385 |
| `r-52` → `r-53` (Linux) | Rebuilds LeBot and three Seer bots | 1,588,164 → 1,588,196 | 1.679 → 1.043 |
| `r-52` → `r-53` (Windows) | Rebuilds LeBot and three Seer bots | 1,583,539 → 1,583,598 | 1.447 → 0.913 |

The archive length hint saves time on small patches. It changes the measured patch sizes by −119 to +154 bytes. Lowering the archive compression level did not preserve size across the test set. For example, the Zstandard level 16 trial with long-distance matching made the `r-24` → `r-25` patch about 178,000 bytes larger.

The unchanged Linux GUI library (`libbob_diff.a`) applied all 11 new patches. The results matched the target archives in paths, file contents, and executable status. We did not run the Windows GUI library. V3 does not record full Unix modes, owners, or timestamps.

## Repeat a case

Run these commands from the workspace root with `gh`, GNU `time`, and `tar` installed. Use a clean directory under `target` for the archives. Set `before` to the path of a release binary from `44af5de` to compare generation times.

```sh
cargo build --release --locked -p bob-cli
mkdir -p target/diff-benchmark
for tag in r-27 r-28; do
    dir="target/diff-benchmark/$tag"
    mkdir -p "$dir"
    gh release download "$tag" --repo RLBot/botpack \
        --pattern botpack_x86_64-linux.tar.xz --dir "$dir"
    tar -xJf "$dir/botpack_x86_64-linux.tar.xz" --strip-components=1 -C "$dir"
    rm "$dir/botpack_x86_64-linux.tar.xz"
done

old=target/diff-benchmark/r-27
new=target/diff-benchmark/r-28
RUST_LOG=off /usr/bin/time -f '%e s' target/release/bob diff "$old" "$new" > target/diff-benchmark/after.bobdiff
wc -c target/diff-benchmark/after.bobdiff

cp -a "$old" target/diff-benchmark/patched
target/release/bob diff-apply target/diff-benchmark/patched < target/diff-benchmark/after.bobdiff
diff -qr "$new" target/diff-benchmark/patched
```

To measure the earlier generator, repeat the timed command with `"$before"` in place of `target/release/bob`. To test the GUI library, link a test driver against `gui/bob-diff/libbob_diff.a`. Pass the patch bytes and a fresh copy of the old tree to `diff_apply`. Check the return code before you compare the trees.
