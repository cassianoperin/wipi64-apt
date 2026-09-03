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

## 1. Publish

```bash
gh release create v0.24 wipi_0.24-0_trixie_arm64.deb \
   --repo cassianoperin/wipi64-apt \
   --notes "WiPi64 0.24"
```

To add or replace a `.deb` on a release that already exists:

```bash
gh release upload v0.24 wipi_0.24-0_trixie_arm64.deb \
   --repo cassianoperin/wipi64-apt --clobber
```

Either action triggers the workflow.

## 2. Check the package version

The only thing APT compares is the `Version` field inside the package. The
release tag and the filename are cosmetic. The scheme in use is
`<upstream>-<revision>`, e.g. `0.23-0`, and the file is named
`wipi_<version>_trixie_arm64.deb`. Keep the two in sync — the `trixie` part
belongs in the filename only, never in the `Version` field, or version
ordering breaks on the next Debian release.

```bash
dpkg-deb -f wipi_0.24-0_trixie_arm64.deb Package Version Architecture
dpkg --compare-versions 0.24-0 gt 0.23-0 && echo "upgrade ordering ok"
```

If the version is unchanged or lower than what is installed, APT silently
offers no upgrade — no error anywhere. This is the most common reason a
"published" update never reaches the Pis, so it is worth checking every time.

Bump the revision (`0.23-0` to `0.23-1`) for packaging-only fixes; bump the
upstream part for actual WiPi changes.

Also worth a look when the file list changed:

```bash
dpkg-deb -c wipi_0.24-0_trixie_arm64.deb   # what gets installed where
dpkg-deb -I wipi_0.24-0_trixie_arm64.deb   # control fields and declared conffiles
```

Anything in the content listing that is *not* a declared conffile gets
overwritten without warning on upgrade.

## 3. Check the workflow ran

```bash
gh run list --repo cassianoperin/wipi64-apt --limit 3
gh run watch --repo cassianoperin/wipi64-apt
```

Or open https://github.com/cassianoperin/wipi64-apt/actions

In the "Baixa os .deb" step log, confirm the line reading
`wipi <version> arm64` shows the version you expect.

A yellow annotation about Node.js 20 comes from a pinned dependency inside
`upload-pages-artifact` and can be ignored.

## 4. Check the published repository

```bash
curl -s https://cassianoperin.github.io/wipi64-apt/Packages | grep -E '^(Package|Version|Filename)'
```

The new version must appear. `InRelease` should also be reachable:

```bash
curl -sI https://cassianoperin.github.io/wipi64-apt/InRelease | head -1
```

Pages can take a few minutes after the workflow finishes.

## 5. Check from a Pi

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

## Fixing a bad release

Deleting the release republishes the index without it, and the package
disappears from APT:

```bash
gh release delete v0.24 --repo cassianoperin/wipi64-apt --cleanup-tag --yes
```

Pis that already upgraded are not rolled back — APT never downgrades on its
own. To move them back, publish a new release with a *higher* version
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
