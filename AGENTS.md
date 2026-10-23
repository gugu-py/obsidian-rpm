# AGENTS.md — Obsidian RPM packaging

Read this before touching `obsidian.spec`. This repo is a single-spec "dist-git lite" for
packaging Obsidian's **official prebuilt Linux x86_64 tarball**. Nothing is compiled; the
spec only repacks upstream's Electron bundle, adds a `.desktop` file, icon and symlinks.

- Current version: **1.14.5** (latest upstream at time of writing)
- Target: Fedora 44 / x86_64, rpm 6.0.2
- Upstream releases: https://github.com/obsidianmd/obsidian-releases/releases
- Copr: https://copr.fedorainfracloud.org/coprs/frieren-blue/obsidian/ (built by CI, see
  "Automation")

Repo layout:

```
obsidian.spec                 # the only artifact that matters; everything hangs off this
AGENTS.md                     # this file
.gitignore                    # keeps *.tar.gz / SRPMs / RPMS out of git
.copr/Makefile                # Copr `make_srpm` entry point: expands Source0 + curls it
.github/workflows/build.yml   # daily upstream check -> spec bump -> Copr SCM build
```

## Golden rules

1. **`obsidian.spec` is the only source artifact.** Never commit the tarball (~131 MB);
   `Source0` is an upstream URL and `.gitignore` already excludes `*.tar.gz`, `*.rpm`,
   `/SOURCES/`, `/RPMS/`, `/SRPMS/`.
2. **Never guess the version.** Always resolve it (see step 1 below). Do not bump the spec
   because a blog post said so. The CI job does this bump on its own — do not fight it.
3. `Version:` **must** equal the tarball version. Reset `Release:` to `1%{?dist}` on every
   version bump and keep exactly one `%changelog` entry per bump.
4. Keep `%global debug_package %{nil}` at the top. The payload contains ELF binaries
   (`obsidian`, `*.so`); without it rpmbuild produces a broken/empty `-debuginfo`.
5. Keep `AutoReqProv: no`. Upstream ships its own libs, so rpm's auto-dependency generator
   would add bogus soname requirements. Consequence: **the `Requires:` list is maintained
   by hand** — see "Dependency check".
6. Keep `ExclusiveArch: x86_64`, **never** `BuildArch: x86_64`. `BuildArch` makes
   `rpmbuild -bs` abort with `No compatible architectures found for build` on Copr, because
   Copr builds SRPMs on **aarch64** builders. `ExclusiveArch` still refuses binary builds on
   non-x86_64 chroots (`Architecture is not included: aarch64`), which is the part that
   matters, while letting the arch-independent SRPM be created anywhere.
7. Changelog date format: `date '+%a %b %d %Y'` in the `C` locale (e.g. `Fri Oct 23 2026`),
   entry `* <date> <Name> <<email>> - <version>-<release>`.
8. Do **not** `sudo dnf install` on the user's machine from an agent session — sudo needs a
   password. Build, verify, then print the install command for the user.

## Version-update runbook

```bash
cd /path/to/repo

# 1. Resolve the real latest tag (GitHub API is often rate-limited → use the redirect)
LATEST=$(curl -fsSL -o /dev/null -w '%{url_effective}' \
  https://github.com/obsidianmd/obsidian-releases/releases/latest | sed 's|.*/releases/tag/v||')
echo "latest: $LATEST"

# 2. Fetch the official tarball into the SOURCES dir, under the *exact basename the spec
#    expects* (obsidian-<ver>.tar.gz). spectool derives it from Source0 for you:
cp obsidian.spec ~/rpmbuild/SPECS/
cd ~/rpmbuild/SPECS && spectool -g -R obsidian.spec   # -> ~/rpmbuild/SOURCES/obsidian-<ver>.tar.gz

# 3. Edit the spec: Version, reset Release, add changelog entry
sed -i "s/^Version:.*/Version:        $LATEST/" obsidian.spec
sed -i 's/^Release:.*/Release:        1%{?dist}/' obsidian.spec
# then add the changelog entry by hand (or `rpmdev-bumpspec -n $LATEST -c "- Update to Obsidian $LATEST" obsidian.spec`)

# 4. Inspect the archive for layout changes *before* building (see "Archive invariants")
tar tzf ~/rpmbuild/SOURCES/obsidian-$LATEST.tar.gz | grep -v '/locales/' | head -40

# 5. Build
rpmbuild -ba ~/rpmbuild/SPECS/obsidian.spec
```

### Source naming (do not "fix" it)

`Source0` is the **upstream URL**, so its basename is lowercase: `obsidian-<version>.tar.gz`.
The tarball's top directory is also lowercase (`obsidian-<version>/`), which is what
`%autosetup -n obsidian-%{version}` matches.

- file in `SOURCES/` → `obsidian-1.14.5.tar.gz` (must match the URL basename)
- **not** `Obsidian-1.14.5.tar.gz` — that was the old form when `Source0` was a bare filename.
  If a stale capital-O file is lying around, rpmbuild will not find the source.

The URL form is deliberate: Copr downloads the source during SRPM creation, so the 131 MB
archive never enters git and never has to be uploaded. `spectool -g -R` gives you the same
behaviour locally. Pinning the URL to `v%{version}` also means the only thing a release bump
has to touch is `Version:`.

### Archive invariants to re-check every release

- Top-level dir is `obsidian-<version>/` (else `%autosetup -n` breaks).
- `obsidian` executable, `resources/`, `resources/icon.png`, `locales/`, `LICENSE.electron.txt`,
  `LICENSES.chromium.html` still exist (they are referenced by `%install`/`%files`).
- **New binaries**: 1.14.x added `obsidian-cli` (now symlinked into `%{_bindir}`). If a new
  top-level tool appears, add a symlink + `%files` entry.
- `*.so` set changed? (e.g. `libffmpeg.so`, `libEGL.so`) → redo the dependency check.
- The archive still must **not** contain a `.desktop` file or scalable icon; the spec
  creates both itself.

## Verify before handing the RPM over

```bash
R=~/rpmbuild/RPMS/x86_64/obsidian-<ver>-1.fc44.x86_64.rpm
rpm -qip $R | head -12          # Version, Size (~378 MB installed), License, Build Date
rpm -qlp $R                     # expect: /usr/bin/obsidian{,-cli}, /usr/share/obsidian/*,
                                #         /usr/share/applications/md.obsidian.Obsidian.desktop,
                                #         /usr/share/icons/hicolor/512x512/apps/obsidian.png,
                                #         /usr/share/licenses/obsidian/*
rpm -qpl $R | grep -c '/locales/'   # locales are packaged (in %{_datadir}/obsidian/*)
```

Functional check without touching the user's config: install a copy into a scratch root
(`dnf install --installroot=...`) or, after a real install, launch once and read
`~/.config/obsidian/obsidian.log` — it must say `Loaded main app package
/usr/share/obsidian/resources/obsidian.asar` and `App is up to date.` / the right version.
`obsidian` accepts **no** `--version` flag; it just starts the GUI (it will hang a shell
command — don't wrap it in a script that waits for exit).

## Dependency check (`AutoReqProv: no`)

```bash
objdump -p /usr/share/obsidian/obsidian | grep NEEDED | awk '{print $2}' | sort -u
for s in <sonames>; do rpm -q --whatprovides "$s()(64bit)"; done
```

Current mapping (Fedora 44) — extend `Requires:` if upstream links something new:

| soname | package | in spec? |
|---|---|---|
| libgtk-3.so.0 | gtk3 | yes |
| libasound.so.2 | alsa-lib | yes |
| libX11.so.6, libxcb.so.1, libXext.so.6, libXcomposite.so.1, libXdamage.so.1, libXfixes.so.3, libXrandr.so.2 | libX11, libxcb, libXext, libXcomposite, libXdamage, libXfixes, libXrandr | partial (`libX11` only) |
| libnss3.so, libnssutil3.so, libsmime3.so, libnspr4.so | nss, nspr | `nss` only |
| libatk-1.0.so.0, libatk-bridge-2.0.so.0, libatspi.so.0 | atk, at-spi2-atk, at-spi2-core | no |
| libcairo.so.2, libpango-1.0.so.0, libglib-2.0.so.0, libgio-2.0.so.0, libgobject-2.0.so.0 | cairo, pango, glib2 | no (pulled by gtk3) |
| libcups.so.2 | cups-libs | no |
| libdbus-1.so.3 | dbus-libs | no |
| libexpat.so.1 | expat | no |
| libgbm.so.1 | mesa-libgbm | no |
| libudev.so.1 | systemd-libs | no |
| libxkbcommon.so.0 | libxkbcommon | no |
| libffmpeg.so, libEGL.so, libGLESv2.so, libvulkan.so.1 | bundled in `%{_datadir}/obsidian` | n/a |

`hicolor-icon-theme` is only needed for the icon cache. `libX11` etc. are only needed on
X11/XWayland; Wayland still loads them indirectly. Keeping the list minimal is deliberate
(these packages are present on any GUI Fedora install), but note the RPM will not pull
them in on a minimal system — that is the trade-off of `AutoReqProv: no`.

## Known pitfalls

- **Desktop-ID collision with Flatpak.** The spec installs
  `/usr/share/applications/md.obsidian.Obsidian.desktop`, which is the *same ID* as
  Flathub's `md.obsidian.Obsidian`. Flatpak export dirs come earlier in `XDG_DATA_DIRS`,
  so if the Flatpak is installed, the launcher and `x-scheme-handler/obsidian` resolve to
  the Flatpak, not to this RPM. Do not "fix" this by renaming the ID casually — the ID is
  what makes Obsidian's own URI handler and `StartupWMClass` matching work. Options: drop
  the Flatpak, or copy the desktop file to `~/.local/share/applications/` (user priority).
- **Copr SCM builds need `.copr/Makefile` and an aarch64-safe spec.** Two things that cost
  real debugging time:
  - With `--method make_srpm`, Copr runs `make -f .copr/Makefile srpm outdir=… spec=…`.
    The `srpm-build` chroot contains **curl + rpm-build only** — no `rpmdevtools`/`spectool`.
    Hence the Makefile expands `Source0` with `rpmspec -P` and downloads it with `curl`.
  - Copr's SRPM builders are **aarch64**, so `BuildArch: x86_64` breaks SRPM creation
    (`No compatible architectures found for build`). Use `ExclusiveArch: x86_64`.
  - The SRPM is created once, so `%{?dist}` is fixed by the SRPM-builder chroot; all chroot
    builds share that release tag. Cosmetic only — EVR comparison ignores the `.fcNN` suffix.
- **Self-updater.** Obsidian's built-in updater writes `obsidian-<ver>.asar` (~27 MB each)
  into the *config dir* (`~/.config/obsidian`) and never replaces the RPM payload. It is
  wasted space/confusing; tell users to disable *Settings → General → Automatic updates*
  and upgrade via a rebuilt RPM.
- **Chromium sandbox.** `chrome-sandbox` is shipped non-setuid; on some systems Electron
  refuses to start. Workaround is `obsidian --no-sandbox` (or a wrapper).
  **Do not** `chmod 4755` it in `%post` — that ships a setuid-root binary in a package.
- **Config vs vaults.** App state lives in `~/.config/obsidian` (Electron/Chromium +
  `obsidian.json` vault list). Vault content, plugins, themes and snippets live inside each
  vault's `.obsidian/` and are never touched by packaging. Flatpak keeps its state in
  `~/.var/app/md.obsidian.Obsidian/config/obsidian`; migrations copy `obsidian.json`,
  per-vault `<vaultid>.json`, `IndexedDB`, `Local Storage`, `Custom Dictionary.txt`,
  `Dictionaries`, `Preferences`, `Cookies`, `user-flags.conf` — never `Cache`,
  `Code Cache`, `GPUCache`, `Partitions`, `*.asar`, `obsidian.log`.
- **Never run the Flatpak and the RPM against the same vault simultaneously** — both write
  `.obsidian/workspace.json`.
- `%{?dist}` means builds are `…-1.fc44`; that is expected for local/Copr builds, not a bug.
- `~/.rpmmacros` must be a **file**; if it is a directory, `rpmdev-setuptree` errors out.
  Delete the stray directory (it happened once on this machine).
- `%post`/`%postun` call `gtk-update-icon-cache` and `update-desktop-database` with
  `|| :` — keep them non-fatal.

## Automation (implemented)

Upstream checks, spec bumps and Copr builds are fully automated. Nothing in this section
should be re-invented; extend it instead.

**Copr project:** https://copr.fedorainfracloud.org/coprs/frieren-blue/obsidian/
(chroots `fedora-43-x86_64`, `fedora-44-x86_64`, `fedora-45-x86_64`, network enabled so the
remote `Source0` can be fetched).

**Workflow:** `.github/workflows/build.yml` runs on a daily cron (05:17 UTC), on pushes that
touch `obsidian.spec`, and on manual dispatch. It:

1. resolves the newest tag via the `releases/latest` **redirect** (the API is rate-limited;
   a 403 here is normal and must not break the job),
2. if newer than `Version:`, rewrites `Version`, resets `Release`, prepends a `%changelog`
   entry, commits and pushes to `main`,
3. triggers `copr-cli buildscm` against the resulting commit — `--method make_srpm`, so Copr
   fetches the 131 MB tarball itself. Nothing large is uploaded or committed.

The Copr credentials live in the repo secret `COPR_CONFIG` (verbatim copy of the local
`~/.config/copr`). To rotate: `gh secret set COPR_CONFIG < ~/.config/copr`.

Manual operations:

```bash
# rebuild the current spec without waiting for the cron
gh workflow run bump-and-build

# rebuild a specific ref / side tag from a workstation (needs ~/.config/copr)
copr-cli buildscm obsidian --clone-url https://github.com/gugu-py/obsidian-rpm \
  --commit main --subdir . --spec obsidian.spec --method make_srpm

# emergency fallback: upload a locally built SRPM (slow, ~100 MB)
copr-cli build obsidian ~/rpmbuild/SRPMS/obsidian-<ver>-1.fc44.src.rpm
```

Notes: pushes made with `GITHUB_TOKEN` do not re-trigger the workflow, which is what stops
the bump→build loop from recursing. `%{?dist}` resolves per chroot, so the same spec yields
`.fc43`/`.fc44`/`.fc45` builds.

Licensing note: Obsidian is proprietary. The tarball is publicly redistributable per its
EULA for personal use, but treat Copr/Fedora "nonfree" expectations conservatively and keep
`License: Commercial`. Packaging it is not a Fedora-review candidate.

## Environment notes (this machine)

Fedora 44, rpm 6.0.2, `rpmbuild`/`rpmdevtools`/`spectool` installed. `copr-cli` is **not**
installed system-wide; use the venv at `~/.local/venvs/copr/bin/copr-cli`, credentials in
`~/.config/copr` (Copr user `frieren-blue`). Wayland/Hyprland session, Flatpak Obsidian may
be present (see collision above). `sudo` requires a password, so agent sessions cannot
install packages.
