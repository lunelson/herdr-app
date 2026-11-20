# Fork notes (herdr-app)

This file lives only on the `fork` branch of lunelson/herdr-app. Never commit it to `master` or a PR branch, and leave it out when cherry-picking patches for upstream. Read it with `AGENTS.md`: it adds to those rules and never replaces them. Machine-specific notes (what is installed where, and when) go in the gitignored `AGENTS.local.md`.

## What the fork is

A machine running the fork has a Herdr.app built from this checkout's `fork` branch instead of an official release. It's built with `scripts/install-macos` and moved to `/Applications` (see Building), so it's signed ad hoc, and `~/.local/bin/herdr-app` points at it. It reports the version of the upstream code it is based on. Builds from source never check for or install updates (`src/update.rs`: only release builds set `HERDR_APP_RELEASE`), wherever the bundle lives.

It's paired with a fork build of herdr-core from the sibling checkout `../herdr-core` (branch `fork`; see its `FORK.md`). Some patches need both halves.

## Keep only one Herdr.app

While on the fork, don't keep an official copy of the app on the machine. Both copies have the bundle ID `dev.herdr.app`, and macOS starts apps by bundle ID (Dock, Spotlight, `open -a Herdr`, and System Settings' "Quit & Reopen" after a privacy permission change), so it may pick either. If the official copy starts, it points `~/.local/bin/herdr-app` at itself and installs its own update, and the first start after such an update runs `hc update`, which replaces the fork core. Check with `lsregister -dump | rg '^path:.*/Herdr\.app'`: only `/Applications/Herdr.app` should show.

## Branches

- `fork`: `upstream/master` plus the patches below and this file, each its own commit. It tracks `origin/fork` on lunelson/herdr-app; it is rebased onto `upstream/master` by naming that explicitly. Only local builds run from it. Never open a PR from it.
- PR branches (e.g. `fix/display-palette`): cut from `upstream/master`, carrying one patch each. Upstream work happens on branches like these, never on `fork`.
- `master`: tracks upstream; keep it untouched.

Remotes: `origin` is lunelson/herdr-app, `upstream` is herdrdev/herdr-app. Every local branch is pushed to `origin` and tracks it, PR branches too, whether or not their PR is open (`git push -u origin <branch>`).

## Patches carried on `fork`

| Patch | Upstream | Status |
|---|---|---|
| `fix: tell the core the theme's palette so programs see its colors (#827)`: display facts sent to the core include the theme's 16 ANSI colors, so programs asking for them (OSC 4, e.g. Pi's default theme) get the theme's colors. Needs the core patch in `../herdr-core`. | issue [herdrdev/herdr-app#827](https://github.com/herdrdev/herdr-app/issues/827); PR branch `fix/display-palette` | diagnosis posted on #827 2026-11-13, waiting on Can; PR not opened yet |
| `feat: layout blocks can fit their content with size {"fit": true}`: a third layout size, so a block in a row (column) takes its content's width (height); used by herdr-mods' `ln-main` tab bar. App only. | none yet; it grows the app's core (layout documents), so it needs Can's go before a PR (`AGENTS.md`) | on `fork` only |

Update this table when a patch is added, lands upstream, or is dropped.

## Setting up a new machine

1. Clone lunelson/herdr-app and lunelson/herdr-core side by side, add `upstream` remotes (herdrdev/…), and check out `fork` in both.
2. Create `AGENTS.local.md` in each checkout (gitignored) with a line saying to read `FORK.md`, plus anything specific to that machine.
3. Install Zig 0.16 (see Building).
4. Install the core first (`../herdr-core/FORK.md`), then the app (Building). Remove any official Herdr.app first (Keep only one Herdr.app).
5. Verify (below).

## Building

Builds need Zig 0.16, installed globally through mise (`zig@0.16.0`). Inside hc terminals the `just` mise shim may find no version, so call `bash scripts/install-macos` directly. Never run a bare `target/*/herdr-app` without a scratch `HERDR_APP_HOME` (`AGENTS.md`).

`scripts/install-macos` always writes `~/Applications/Herdr.app`. Move the result over `/Applications/Herdr.app` so only one Herdr.app exists (a running Herdr keeps going; it picks up the new build when reopened):

```sh
bash scripts/install-macos
rm -rf /Applications/Herdr.app && mv ~/Applications/Herdr.app /Applications/Herdr.app
LSR=/System/Library/Frameworks/CoreServices.framework/Frameworks/LaunchServices.framework/Support/lsregister
"$LSR" -u ~/Applications/Herdr.app; "$LSR" -f /Applications/Herdr.app
ln -sfn /Applications/Herdr.app/Contents/MacOS/Herdr ~/.local/bin/herdr-app
```

## Updating `fork` when upstream moves

Update both forks in one go, core first, so the app never runs against a core without its half of a patch. To see what's new upstream, run in each checkout:

```sh
git fetch upstream && git log --oneline fork..upstream/master
```

1. If herdr-core's upstream moved, update the core fork first (`../herdr-core/FORK.md`). That restarts the user's real core; ask first.
2. `git switch fork && git rebase upstream/master`
3. If a carried patch has landed upstream, it was squash-merged, so it won't match: the rebase stops with conflicts or empty commits. Drop the carried commit (`git rebase --skip`, or `git rebase -i` and delete it) and update the table above.
4. Run `just test` (what CI runs). If `just` can't run, run its two lines directly: the `bun test scripts/…` list and `cargo test --locked`.
5. Reinstall (see Building), then quit Herdr and reopen `/Applications/Herdr.app`. Ask the user before quitting their app.
6. Verify (below).
7. `git push --force-with-lease origin fork`. After a rebase, `fork` and `origin/fork` have diverged on purpose: don't `git pull`, which would merge the old history back in. On another machine, take the pushed branch with `git fetch origin && git reset --hard origin/fork` (only when that machine's `fork` has no commits of its own).

Also rebase PR branches on `upstream/master` and push them (`git push --force-with-lease origin <branch>`).

## Verify after a reinstall

- `herdr-app --version` reports the upstream version the fork is now based on, and `pgrep -lf Herdr.app` shows it running from `/Applications/Herdr.app`.
- `codesign -dv /Applications/Herdr.app` and `codesign -dv "$(readlink -f ~/.local/bin/hc)"` both say `Signature=adhoc`: the app and core forks are in place, not official builds.
- `lsregister -dump | rg '^path:.*/Herdr\.app'` lists only `/Applications/Herdr.app`.
- The palette patch works: Pi with its default theme shows the Herdr theme's colors.
- The personal mods still load on the new version: `just check` in `../herdr-mods`, then `herdr-app status` shows no errors.

## Going back to the official app

Once official releases carry every patch the machine depends on:

1. Quit Herdr and delete `/Applications/Herdr.app` (the fork).
2. Install the official app from herdr.dev into `/Applications`. Opening it points `~/.local/bin/herdr-app` at itself.
3. Let it update if it offers one. Then `hc update` for the official core (`../herdr-core/FORK.md`).
4. On that machine, delete `AGENTS.local.md` or mark it retired. Once no machine uses the fork, delete the `fork` branch.
