# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

The parent `../CLAUDE.md` describes what every formula of the collection shares
(toolchain, `map.jinja` merge chain, `_mapdata`, Kitchen, CI, lint rules). This
file only records what is specific to `sudoers-formula`, which departs from the
template in several ways.

## Commands

Two suites, three platforms each: six instances.

```bash
bin/kitchen list
bin/kitchen test default-debian-13-3008-2    # main file managed, full pillar
bin/kitchen test appended-debian-11-3006-27  # main file NOT managed, #include appended
pre-commit run --all-files
```

## Layout — not the template's package/config/service split

- `sudoers/init.sls` (state `sudoers`) does everything for the main file in one
  place: `pkg.installed` for `sudo`, then `/etc/sudoers` rendered from
  `sudoers/files/sudoers`. With `manage_main_config: false` the file is left
  alone and a `test.show_notification` takes its state ID instead.
- `sudoers/included/install.sls` (state `sudoers.included`) includes `sudoers`,
  manages the include directory (`clean:` driven by `purge_includedir`) and
  renders one file per `included_files` entry from the **same**
  `files/sudoers` template, with `included: True`. A key without `/` is a file
  name inside `includedir`; a key with `/` is an absolute path.
- When the main file is not managed and `append_included_files_to_endof_main_config`
  is true, each included file is added to the existing `/etc/sudoers` as an
  `#include <path>` line through `file.append` (sudo applies the last match, so
  this guarantees the included rules win).
- Every generated file goes through `check_cmd: visudo -c -f`: a template change
  that produces invalid sudoers syntax fails the state instead of locking sudo.
- No TOFS: there is no `libtofs.jinja`, sources are literal `salt://sudoers/...`
  paths.

## Two configuration paths — the main pitfall

`map.jinja` only supplies the structural keys (`pkg`, `configpath`, `group`,
`execprefix`, `includedir`, `included_files`, the booleans). The **content** of
`/etc/sudoers` does not go through it: `files/sudoers` reads
`pillar.get('sudoers', {})` directly when `included` is false. Consequences:

- `sudoers:lookup` and the `os*map.yaml` files have no effect on the rules
  written to `/etc/sudoers`.
- On `os_family: Debian`, the template falls back to Debian-like defaults
  (`env_reset`, `mail_badpass`, `secure_path`, `root` and `%sudo` with
  `ALL=(ALL:ALL) ALL`) only for the pillar keys that are **absent**. Setting
  `sudoers:defaults` or `sudoers:users` replaces the matching fallback entirely;
  nothing is merged with Debian's packaged `/etc/sudoers`.
- The `#includedir` line is written as the legacy `#includedir` form, taken from
  the pillar `includedir` (default `/etc/sudoers.d` for the main file, none for
  included files).

For included files, the template gets the per-file spec as its `sudoers`
context, so each entry of `included_files` accepts the same `users`, `groups`,
`netgroups`, `defaults` and `aliases` keys as the top-level pillar.

## Unused or vestigial pieces

- `sudoers/macros.sls` defines a `sudoers()` Jinja macro (one
  `/etc/sudoers.d/<priority>-<user>-<sudo>` file rendered from
  `templates/sudoers.jinja`). No state of the formula imports it; it exists for
  external states to `import`.
- `sls_config_file` in `init.sls` and `included/install.sls` points at a
  `sudoers.config.file` state that does not exist; it is never used.
- `osarchmap.yaml` sets `arch`, and `map.jinja` sets `rootgroup` on macOS; no
  state reads either.
- `test/integration/*/files/_mapdata/` still holds upstream's references for
  many distributions; only `debian-11.yaml` and `debian-13.yaml` are exercised.
  Both suites merge `test/salt/pillar/kitchen.sls` (the `kitchen` NOPASSWD user
  InSpec needs), so that user appears in the reference dumps too.

## Context

`AUDIT.md` (untracked) is the 2026-10-17 audit of this fork: how it is used in
production (`sudoers.included` on every host, `included_files` unset), the Debian 13
`use_pty` default that the generated `/etc/sudoers` drops, and the
`map.jinja` import reduction it recommends.
