# obsidian-rpm

Fedora RPM packaging for [Obsidian](https://obsidian.md), built from the **official upstream
x86_64 tarball** (no compilation — it repacks Electron, adds a `.desktop` file, icon and
`/usr/bin` symlinks).

## Install

```bash
sudo dnf copr enable frieren-blue/obsidian
sudo dnf install obsidian
```

Builds are available for Fedora 43, 44 and 45 (x86_64).

## Updating

Nothing to do manually in the normal case. A GitHub Actions job checks upstream daily,
bumps `obsidian.spec` (`Version`, `Release`, `%changelog`) and triggers a Copr SCM build.
To force a rebuild: `gh workflow run bump-and-build`, or use the *Builds* tab on Copr.

Local build:

```bash
rpmdev-setuptree
cp obsidian.spec ~/rpmbuild/SPECS/
spectool -g -R ~/rpmbuild/SPECS/obsidian.spec
rpmbuild -ba ~/rpmbuild/SPECS/obsidian.spec
```

Full maintenance policy, pitfalls and verified dependency notes: see **[AGENTS.md](AGENTS.md)**.

## Notes

- Obsidian is proprietary software (`License: Commercial`); this repo only repackages the
  publicly distributed tarball. It is not a Fedora-review candidate.
- Disable *Settings → General → Automatic updates* in Obsidian: its built-in updater writes
  `.asar` files into `~/.config/obsidian` and cannot replace an RPM payload. Upgrade via dnf.
- The desktop file ID is `md.obsidian.Obsidian.desktop`, the same as Flathub's package. If
  the Flatpak is also installed, launchers may prefer it — remove the Flatpak or copy the
  desktop file into `~/.local/share/applications/`.
