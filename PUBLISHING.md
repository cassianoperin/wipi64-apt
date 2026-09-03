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

## 1. Check the package before publishing

The only thing APT compares is the `Version` field inside the package. The
release tag and the filename are cosmetic.

```bash
dpkg-deb -f wipi_0.24_arm64.deb Package Version Architecture
dpkg --compare-versions 0.24 gt 0.23 && echo "upgrade ordering ok"
```

If the version is unchanged or lower than the installed one, APT will silently
offer no upgrade — the most common reason a "published" update never reaches
the Pis.

Also worth a look, especially if the file list changed:

```bash
dpkg-deb -c wipi_0.24_arm64.deb   # what gets installed where
dpkg-deb -I wipi_0.24_arm64.deb   # control fields and declared conffiles
```

Anything in the content listing that is *not* a declared conffile gets
overwritten without warning on upgrade.

## 2. Publish

```bash
gh release create v0.24 wipi_0.24_arm64.deb \
   --repo cassianoperin/wipi64-apt \
   --notes "WiPi64 0.24"
```

To add a `.deb` to a release that already exists:

```bash
gh release upload v0.24 wipi_0.24_arm64.deb \
   --repo cassianoperin/wipi64-apt --clobber
```

Either action triggers the workflow.

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
sudo apt update
apt policy wipi
```

Expected: the new version listed as a candidate, with
`cassianoperin.github.io` as its origin. Then:

```bash
sudo apt install --only-upgrade wipi
dpkg -l wipi
```

## Fixing a bad release

Deleting the release republishes the index without it, and the package
disappears from APT:

```bash
gh release delete v0.24 --repo cassianoperin/wipi64-apt --cleanup-tag --yes
```

Pis that already upgraded are not rolled back. To move them back, publish a
new release with a higher version containing the older code — APT never
downgrades on its own.

## Notes

- Only the 5 most recent releases are indexed. Older ones stay on GitHub but
  drop out of APT.
- To re-publish the index without creating a release (for example after
  editing the workflow): Actions → publish-apt → Run workflow, or
  `gh workflow run publish-apt --repo cassianoperin/wipi64-apt`.
