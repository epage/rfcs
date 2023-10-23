- Feature Name: `split-home`
- Start Date: 10/23/2023
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
providing environment variables to control each of these types of paths

# Motivation
[motivation]: #motivation

Benefits include:
- Using a `.cargo` directory is violating the recommendations/rules on most
  operating systems. _(Linux, Windows, macOS. Especially painful on Windows,
  where dotfiles are not hidden.)_
- Putting caches in designated cache directories allows backup tools to ignore
  them. _(Linux, Windows, macOS. Example: Time Machine ignores the cache
  directory on macOS.)_
- It makes it easier for users to manage, share and version-control their
  configuration files, as configuration files from different applications end up
  in the same place, instead of being intermingled with cache files. _(Linux
  and macOS.)_
- Cargo contributes to the slow cleanup of the `$HOME` directory by stopping to
  add its application-private clutter to it. _(Linux.)_
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
- `CARGO_CACHE_HOME`
- `RUSTUP_CONFIG_HOME`
- `RUSTUP_CACHE_HOME`

Previously, we had `CARGO_HOME` and `RUSTUP_HOME` which allowed moving the directories as a monolith.

A Linux user could run the following migration script
```bash
CARGO_HOME=$(realpath ~/cargo)
CARGO_CONFIG_HOME=$(realpath ~/.config/cargo)
mkdir -p $CARGO_CONFIG_HOME
echo CARGO_CONFIG_HOME=$CARGO_CONFIG_HOME >> ~/.bashrc
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
migrate_cargo_cache credentials.toml  # avoid backing up secrets
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
- `cargo_config_homes`: Returns all of
  - `home::cargo_config_home()`
  - `home::cargo_home()`
- `cargo_config_home`: Returns
  - `CARGO_CONFIG_HOME`, if set
- `cargo_cache_home`: Returns the first match
  1. `CARGO_CACHE_HOME`, if set
  2. `home::cargo_home()`
- `rustup_config_homes`: Returns all of
  - `home::rustup_config_home()`
  - `home::rustup_home()`
- `rustup_config_home`: Returns
  - `RUSTUP_CONFIG_HOME`, if set
- `rustup_cache_home`: Returns the first match
  1. `RUSTUP_CACHE_HOME`, if set
  2. Return `home::rustup_home()`

Cargo will be modified to call these more specific home directories,
based on the above migration script.

Each of these new environment variables will be blocked from being set in [`config.toml`s `[env]` table](https://doc.rust-lang.org/cargo/reference/config.html#env).

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The central thesis of this design is that the caches are throw-away so we don't
need to provide a migration path for them but we do for config.
From that, we are copying git's model of layering both the old and new config locations on top of each other

Existing issues:
- [rust-lang/cargo#1734](https://github.com/rust-lang/cargo/issues/1734)
- [rust-lang/rustup#247](https://github.com/rust-lang/rustup/issues/247)

Previous proposals:
- [RFC 1615](ahttps://github.com/rust-lang/rfcs/pull/1615)
- [spacekookie's unpublished RFC](https://github.com/spacekookie/rfcs/pull/1/files)
- [poignardazur's blog post](https://poignardazur.github.io/2023/05/23/platform-compliance-in-cargo/)

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

- Is `credentials.toml` best under `CARGO_CACHE_HOME`?
- Does config fallback work for rustup?
- What to do about `$CARGO_HOME/bin`?
  - Can we defer it?
  - How could we support it?
    - Requires interop between rustup and cargo which ship separately
    - Bins aren't throwaway, like cache
    - For XDG, [the spec was modified](https://gitlab.freedesktop.org/xdg/xdg-specs/-/merge_requests/38) to reference `~/.local/bin` without an `XDG_BIN_HOME` variable
    - Rustup assumes total control of `$CARGO_HOME/bin`
    - Files aren't all under `bin/`:
      - `$CARGO_HOME/.crates2.json`
      - `$CARGO_HOME/.crates.toml`
      - `$CARGO_HOME/env`
      - `$CARGO_HOME/bin`

# Future possibilities
[future-possibilities]: #future-possibilities

## Platform-specific paths

Update [`home` package](https://crates.io/crates/home) with the following
- `cargo_config_home`: Returns
  - `CARGO_CONFIG_HOME`, if set
  - linux or macOS:
    - `$XDG_CONFIG_HOME/cargo`, if set
    - `~/.config/cargo`
  - windows:
    - `AppData\Roaming\Cargo`
- `cargo_cache_home`: Returns the first match
  1. `CARGO_CACHE_HOME`, if set
  2. linux or macOS:
    1. `$XDG_CACHE_HOME/cargo`, if set
    2. `~/.cache/cargo`
  3. windows:
    - `AppData\Local\Temp\Cargo`
- `rustup_config_home`: Returns
  - `RUSTUP_CONFIG_HOME`, if set
  - linux or macOS:
    - `$XDG_CONFIG_HOME/rustup`, if set
    - `~/.config/rustup`
  - windows:
    - `AppData\Roaming\Rustup`
- `rustup_cache_home`: Returns the first match
  1. `RUSTUP_CACHE_HOME`, if set
  2. linux or macOS:
    1. `$XDG_CACHE_HOME/rustup`, if set
    2. `~/.cache/rustup`
  3. windows:
    - `AppData\Local\Temp\Cargo`

macOS: This favors XDG over the platform-specific application directories to be
more consistent with other CLI developer tooling.  The platform-specific
application directories can always be emulated by setting the appropriate
environment variables.

## Cargo CLI for reading the values

While applications can call into `home` to get the values, sometimes a user will want it interactively.

Potential ideas include:
- A `cargo dirs` built-in command
- Reuse `cargo config`
- `cargo --print <target>` much like `rustc --print <target>`
