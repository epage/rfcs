- Feature Name: `split-home`
- Start Date: 10/23/2023
- Pre-RFC: [Internals](https://internals.rust-lang.org/t/pre-rfc-split-cargo-home/19747) ([zulip](https://rust-lang.zulipchat.com/#narrow/stream/246057-t-cargo/topic/Pre-RFC.3A.20Split.20CARGO_HOME))
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Today, `cargo` stores all user-global files in `CARGO_HOME` (`~/.cargo`), including
- Caches
- Config
- Installed binaries

Similarly, `rustup` stores all user-global files in `~/.rustup`.

This RFC would provide a way for users to emulate platform-specific paths by
providing environment variables to control each of these types of paths as an incremental,
transitional step towards eventually supporting platform-specific paths.

# Motivation
[motivation]: #motivation

Benefits include:
- Using a `.cargo` directory is violating the recommendations/rules on most
  operating systems. _(Linux, Windows, macOS. Especially painful on Windows,
  where dotfiles are not hidden)_
- Putting caches in designated cache directories allows backup tools to ignore
  them. _(Linux, Windows, macOS. Example: Time Machine ignores the cache
  directory on macOS)_
- It makes it easier for users to manage, share and version-control their
  configuration files, as configuration files from different applications end up
  in the same place, instead of being intermingled with cache files. _(Linux
  and macOS)_
- Cargo contributes to the slow cleanup of the `$HOME` directory by stopping to
  add its application-private clutter to it. _(Linux)_
- Using a standard directory for binary outputs can allow the user to execute
  Cargo-installed binaries without modifying their `PATH` variable. _(Linux)_

See [arch's wiki](https://wiki.archlinux.org/title/XDG_Base_Directory) for a
non-exhaustive list of software that has native support for XDG paths and
software that allows emulating it, like in this proposal.

While providing full support for XDG paths, and the Windows equivalent, would be ideal,
we are going for this scaled back solution for now because
- We'll likely want a transition window anyways and this just splits each phase into its own RFC, rather than deciding on every phase's details up front
- We can get users benefits now without getting bogged down in the details like whether macOS users should use XDG or OS native paths
- This allows us to collect feedback and iterate, rather than predict how important each use case is (e.g. how to lookup the paths)

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Users who want to split up `~/.cargo` or `~/rustup` into platform-specific paths can set
- `CARGO_CONFIG_HOME`
- `CARGO_DATA_HOME`
- `CARGO_BIN_HOME`
- `CARGO_CONFIG_HOME`
- `RUSTUP_CONFIG_HOME`
- `RUSTUP_CACHE_HOME`

Previously, we had `CARGO_HOME` and `RUSTUP_HOME` which allowed moving the directories as a monolith.

A Linux user could run the following migration script
```bash
CARGO_HOME=$(realpath ~/cargo)
CARGO_CONFIG_HOME=$(realpath ~/.config/cargo)
mkdir -p $CARGO_CONFIG_HOME
echo CARGO_CONFIG_HOME=$CARGO_CONFIG_HOME >> ~/.bashrc
CARGO_DATA_HOME=$(realpath ~/.local/share/cargo)
mkdir -p $CARGO_DATA_HOME
echo CARGO_DATA_HOME=$CARGO_DATA_HOME >> ~/.bashrc
CARGO_BIN_HOME=$(realpath ~/.local/share/cargo/bin)
echo CARGO_BIN_HOME=$CARGO_BIN_HOME >> ~/.bashrc
CARGO_CACHE_HOME=$(realpath ~/.cache/cargo)
mkdir -p $CARGO_CACHE_HOME
echo CARGO_CACHE_HOME=$CARGO_CACHE_HOME >> ~/.bashrc

RUSTUP_HOME=$(realpath ~/rustup)
RUSTUP_CONFIG_HOME=$(realpath ~/.config/rustup)
mkdir -p $RUSTUP_CONFIG_HOME
echo RUSTUP_CONFIG_HOME=$RUSTUP_CONFIG_HOME >> ~/.bashrc
RUSTUP_CACHE_HOME=$(realpath ~/.cache/rustup)
mkdir -p $RUSTUP_CACHE_HOME
echo RUSTUP_CACHE_HOME=$RUSTUP_CACHE_HOME >> ~/.bashrc

function migrate_cargo_config {
    local item=$1
    mv $CARGO_HOME/$item $CARGO_CONFIG_HOME/$item
    ln -s $CARGO_CONFIG_HOME/$item $CARGO_HOME/$item
}

function migrate_cargo_data {
    local item=$1
    mv $CARGO_HOME/$item $CARGO_DATA_HOME/$item
    ln -s $CARGO_DATA_HOME/$item $CARGO_HOME/$item
}

function migrate_cargo_bin {
    mv $CARGO_HOME/bin $CARGO_BIN_HOME
    ln -s $CARGO_BIN_HOME/$item $CARGO_HOME/$item
}

function migrate_cargo_cache {
    local item=$1
    mv $CARGO_HOME/$item $CARGO_CACHE_HOME/$item
    ln -s $CARGO_CACHE_HOME/$item $CARGO_HOME/$item
}

function migrate_rustup_config {
    local item=$1
    mv $RUSTUP_HOME/$item $RUSTUP_CONFIG_HOME/$item
    ln -s $RUSTUP_CONFIG_HOME/$item $RUSTUP_HOME/$item
}

function migrate_rustup_cache {
    local item=$1
    mv $RUSTUP_HOME/$item $RUSTUP_CACHE_HOME/$item
    ln -s $RUSTUP_CACHE_HOME/$item $RUSTUP_HOME/$item
}

migrate_cargo_config config.toml
migrate_cargo_data env
migrate_cargo_data .crates.toml
migrate_cargo_data .crates2.json
migrate_cargo_data credentials.toml  # avoid backing up secrets
migrate_cargo_bin
migrate_cargo_cache registry
migrate_cargo_cache git
migrate_cargo_cache target  # used by "cargo script"
migrate_cargo_cache .package-cache
migrate_cargo_cache .package-cache-mutate

migrate_rustup_config settings.toml
migrate_rustup_cache downloads
migrate_rustup_cache tmp
migrate_rustup_cache toolchains
migrate_rustup_cache update-hashes
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

We'll add to the confusingly named [`home` package](https://crates.io/crates/home), the following
- `cargo_config_home`: Returns the first match
  1. `CARGO_CONFIG_HOME`, if set
  2. `home::cargo_home()`
- `cargo_data_home`: Returns the first match
  1. `CARGO_DATA_HOME`, if set
  2. `home::cargo_home()`
- `cargo_bin_home`: Returns the first match
  1. `CARGO_BIN_HOME`, if set
  2. `home::cargo_home().join("bin")`
- `cargo_cache_home`: Returns the first match
  1. `CARGO_CACHE_HOME`, if set
  2. `home::cargo_home()`
- `rustup_config_home`: Returns the first match
  1. `RUSTUP_CONFIG_HOME`, if set
  2. `home::rustup_home()`
- `rustup_cache_home`: Returns the first match
  1. `RUSTUP_CACHE_HOME`, if set
  2. Return `home::rustup_home()`

Cargo will be modified to call these more specific home directories,
based on the above migration script.

Each of these new environment variables will be blocked from being set in [`config.toml`s `[env]` table](https://doc.rust-lang.org/cargo/reference/config.html#env).

When discovering a workspace root, `CARGO_CACHE_HOME` will be treated as a stop-path, like `CARGO_HOME`

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

`~/.cargo/bin` and related files requires both updated rustup and cargo

rustup assumes complete control of `~/.cargo/bin` and people might map it to `~/.local/bin` which would cause unexpected behavior

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

`credentials.toml` was put under `CARGO_DATA_HOME` as its program-managed data
- `CARGO_CONFIG_HOME` might cause it to get backed up to public git repos, exposing secrets
- Ideally people will start migrating to [OS-native credential stores](https://doc.rust-lang.org/nightly/cargo/reference/registry-authentication.html)

Existing issues:
- [rust-lang/cargo#1734](https://github.com/rust-lang/cargo/issues/1734)
- [rust-lang/rustup#247](https://github.com/rust-lang/rustup/issues/247)

Previous proposals:
- [RFC 1615](https://github.com/rust-lang/rfcs/pull/1615)
- [spacekookie's unpublished RFC](https://github.com/spacekookie/rfcs/pull/1/files)
- [poignardazur's blog post](https://poignardazur.github.io/2023/05/23/platform-compliance-in-cargo/)

An alternative approach is to rely on caches being throw-away and to instead do
- Read both configs (like git)
- Hard split for cache

Reasons we didn't go with this:
- This doesn't work for `CARGO_BIN_HOME`
- Rustup doesn't support a layered config for reading from two locations at once

# Prior art
[prior-art]: #prior-art

git
- Layers both the old `~/.git/config` and the new `~/config/git/config` on top of each other
- No user cache

ansible
- `ANSIBLE_HOME`
- `ANSIGLE_HOME_CONFIG` (config file path)
- `ANSIBLE_GALAXY_CACHE_DIR`

asdf
- `ASDF_CONFIG_FILE`
- `ASDF_DATA_DIR`

For more, see [arch's wiki entry for XDG Base Directory](https://wiki.archlinux.org/title/XDG_Base_Directory).

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Does config fallback work for rustup?
- What should be done for env variables that are empty (treat is unset?) or relative (unset? error?)?
- Is rustup `update-hashes` cache, state, data, or config?
- Is rustup `toolchains` cache, state, data, or config?
- What all paths should `rustup uninstall` delete?

# Future possibilities
[future-possibilities]: #future-possibilities

## Platform-specific paths

**Note:** The information here is for illustrative purposes and discussing the details
is only relevant so far as it might affect a decision being made within this RFC.

Update [`home` package](https://crates.io/crates/home) with the following
- `cargo_config_home`: Returns the first match
  1. `CARGO_CONFIG_HOME`, if set
  2. `CARGO_HOME`, if set
  3. `cargo_home()`, if present
  4. linux or macOS:
    1. `$XDG_CONFIG_HOME/cargo`, if set
    2. `~/.config/cargo`
  5. windows:
    - `AppData\Roaming\Cargo`
- `cargo_data_home`: Returns the first match
  1. `CARGO_DATA_HOME`, if set
  2. `CARGO_HOME`, if set
  3. `cargo_home()`, if present
  4. linux or macOS:
    1. `$XDG_DATA_HOME/cargo`, if set
    2. `~/.local/share/cargo`
  5. windows:
    - TBD
- `cargo_bin_home`: Returns the first match
  1. `CARGO_BIN_HOME`, if set
  2. `CARGO_HOME/bin`, if set
  3. `cargo_home().join("bin")`, if present
  4. linux or macOS:
    - TBD
  5. windows:
    - TBD
- `cargo_cache_home`: Returns the first match
  1. `CARGO_CACHE_HOME`, if set
  2. `CARGO_HOME`, if set
  3. `cargo_home()`, if present
  4. linux or macOS:
    1. `$XDG_CACHE_HOME/cargo`, if set
    2. `~/.cache/cargo`
  5. windows:
    - `AppData\Local\Temp\Cargo`
- `rustup_config_home`: Returns
  1. `RUSTUP_CONFIG_HOME`, if set
  2. `RUSTUP_HOME`, if set
  3. `cargo_home()`, if present
  4. linux or macOS:
    1. `$XDG_CONFIG_HOME/rustup`, if set
    2. `~/.config/rustup`
  5. windows:
    - `AppData\Roaming\Rustup`
- `rustup_cache_home`: Returns the first match
  1. `RUSTUP_CACHE_HOME`, if set
  2. `RUSTUP_HOME`, if set
  3. `cargo_home()`, if present
  4. linux or macOS:
    1. `$XDG_CACHE_HOME/rustup`, if set
    2. `~/.cache/rustup`
  5. windows:
    - `AppData\Local\Temp\Cargo`

Windows: [Roaming app data is no longer supported on Windows 11](https://learn.microsoft.com/en-us/windows/apps/design/app-settings/store-and-retrieve-app-data#roaming-data).
We'll still use these paths to communicate intent to Windows to be future compatible.

macOS: This currently favors XDG over the platform-specific application directories to be
more consistent with other CLI developer tooling.
The platform-specific application directories can always be emulated by setting
the appropriate environment variables.
The final decision will be left to the relevant RFC

Open questions
- The paths for macOS
- Local vs roaming on Windows
- Should the higher-precedence-with-presence check be used for `cargo_home` or platform-specific paths?
- How do we want to handle the fact that rustup proxies set `CARGO_HOME`?  When we get to the state when users aren't explicitly setting more specific variables, rustup's `CARGO_HOME` will win out

## Automated migration

Add a `rustup` subcommand to migrate people to this

## Compat symlinks

Have rustup manage symlinks when installing older toolchains

## Cargo CLI for reading the values

While applications can call into `home` to get the values, sometimes a user will want it interactively.

Potential ideas include:
- A `cargo dirs` built-in command
- Reuse `cargo config`
- `cargo --print <target>` much like `rustc --print <target>`

## Allow setting `CARGO_CONFIG_HOME` from `.cargo/config.toml`

For sharing of caches between host and docker,
[rust-lang/cargo#6452](https://github.com/rust-lang/cargo/issues/6452)
requested config-control over the global cache location.
Being a cache,
all results should be safe to re-calculate,
unless some of the other `CARGO_*_HOME` variables.
