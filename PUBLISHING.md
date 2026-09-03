# Publishing a new version

Maintainer guide for `cassianoperin/wipi64-apt`.

Publishing means: attach a `.deb` to a GitHub Release. A workflow rebuilds and
signs the APT index and publishes it to GitHub Pages. There is no server to
touch.

## One-time setup

Authenticate the GitHub CLI on the machine you publish from:

```bash
gh auth login
gh auth status
```

## 1. Update the release notes

`release_notes.txt` in the root of this repository is what the WiPi Updater page
shows to users before they confirm an upgrade. Add the new version at the
**top** of the changelog, newest first:

```
0.24-0 (2026.09.02)
  - What changed
```

Commit and push it. The workflow copies it to
`https://cassianoperin.github.io/wipi64-apt/release_notes.txt`, which is the URL
`update.php` reads.

Forgetting this step means users see the previous version's notes when
deciding whether to upgrade — nothing breaks, but the page lies.

## 2. Publish

```bash
gh release create v0.24 wipi_0.24-0_arm64.deb \
   --repo cassianoperin/wipi64-apt \
   --notes "WiPi64 0.24"
```

To add or replace a `.deb` on a release that already exists:

```bash
gh release upload v0.24 wipi_0.24-0_arm64.deb \
   --repo cassianoperin/wipi64-apt --clobber
```

Either action triggers the workflow.

## 3. Check the package version

The only thing APT compares is the `Version` field inside the package. The
release tag and the filename are cosmetic. The scheme in use is
`<upstream>-<revision>`, e.g. `0.24-0`, and the file is named
`wipi_<version>_arm64.deb` — the canonical Debian form, which is also what APT
uses when it caches the download on a Pi.

Build it with `dpkg-name` so the filename is derived from the control file
rather than typed by hand:

```bash
cd /root/deb
dpkg-deb --build wipi_0.24-0
dpkg-name wipi_0.24-0.deb
```

Never put the Debian suite (`trixie`) in the `Version` field. Version
comparison is character by character, so `0.25-0~forky` would sort *lower*
than `0.25-0~trixie` and the upgrade would silently never be offered.

```bash
dpkg-deb -f wipi_0.24-0_arm64.deb Package Version Architecture
dpkg --compare-versions 0.24-0 gt 0.23-0 && echo "upgrade ordering ok"
```

If the version is unchanged or lower than what is installed, APT silently
offers no upgrade — no error anywhere. This is the most common reason a
"published" update never reaches the Pis, so it is worth checking every time.

Bump the revision (`0.23-0` to `0.23-1`) for packaging-only fixes; bump the
upstream part for actual WiPi changes.

Also worth a look when the file list changed:

```bash
dpkg-deb -c wipi_0.24-0_arm64.deb   # what gets installed where
dpkg-deb -I wipi_0.24-0_arm64.deb   # control fields and declared conffiles
```

Anything in the content listing that is *not* a declared conffile gets
overwritten without warning on upgrade.

## 4. Check the workflow ran

```bash
gh run list --repo cassianoperin/wipi64-apt --limit 3
gh run watch --repo cassianoperin/wipi64-apt
```

Or open https://github.com/cassianoperin/wipi64-apt/actions

In the "Baixa os .deb" step log, confirm the line reading
`wipi <version> arm64` shows the version you expect.

A yellow annotation about Node.js 20 comes from a pinned dependency inside
`upload-pages-artifact` and can be ignored.

## 5. Check the published repository

```bash
curl -s https://cassianoperin.github.io/wipi64-apt/Packages | grep -E '^(Package|Version|Filename)'
```

The new version must appear. `InRelease` should also be reachable:

```bash
curl -sI https://cassianoperin.github.io/wipi64-apt/InRelease | head -1
```

Pages can take a few minutes after the workflow finishes.

## 6. Check from a Pi

On a test Pi — not a production cabinet:

```bash
apt update
apt policy wipi
```

Expected output, with the new version as `Candidate` and
`cassianoperin.github.io` as its origin:

```
wipi:
  Installed: 0.23-0
  Candidate: 0.24-0
  Version table:
     0.24-0 500
        500 https://cassianoperin.github.io/wipi64-apt ./ Packages
 *** 0.23-0 100
        100 /var/lib/dpkg/status
```

Then upgrade:

```bash
apt install --only-upgrade wipi
apt policy wipi
```

`Installed:` should now show the new version, with the `***` marker on the
repository line.

## Troubleshooting

### `Tag "vX.Y" is not allowed to deploy to github-pages`

The `github-pages` environment restricts which refs may deploy, and a release
event runs in the context of its tag rather than a branch. Fix it once at
Settings -> Environments -> `github-pages` -> Deployment branches and tags:
either add a rule with ref type **Tag** and pattern `v*`, or set the dropdown
to **No restriction**.

Until that is done, Run workflow from the Actions tab still works — a manual
run executes on the default branch, which is allowed.

## Fixing a bad release

Deleting the release republishes the index without it, and the package
disappears from APT:

```bash
gh release delete v0.24 --repo cassianoperin/wipi64-apt --cleanup-tag --yes
```

`--cleanup-tag` deletes the tag too, so the same version number can be reused
after fixing the package. Omit it to keep the tag and publish the fix as a new
version instead.

Pis that already upgraded are not rolled back — APT never downgrades on its
own. To move a single Pi back, install the older version explicitly:

```bash
apt install wipi=0.23-0
```

To move everyone back, publish a new release with a *higher* version
containing the older code, e.g. `0.24-1` reverting to the 0.23 contents.

## Notes

- Only the 5 most recent releases are indexed. Older ones stay on GitHub but
  drop out of APT. A specific kept version can still be installed with
  `apt install wipi=0.23-0`.
- To re-publish the index without creating a release (for example after
  editing the workflow): Actions -> publish-apt -> Run workflow, or
  `gh workflow run publish-apt --repo cassianoperin/wipi64-apt`.
- Client setup for a Pi that does not have the repository configured yet is in
  the README.
- `release_notes.txt` is copied into the published site by the workflow. If the
  WiPi Updater page shows nothing under "Release Notes", check that
  `https://cassianoperin.github.io/wipi64-apt/release_notes.txt` is reachable.
