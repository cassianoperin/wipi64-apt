# wipi64-apt

APT repository for the `wipi` package used by
[WiPiNetbooter64](https://github.com/cassianoperin/WiPiNetbooter64).

Once configured, WiPi updates are installed with `apt` — no manual `.deb`
download, no SD card reflash.

## Installation

On the Raspberry Pi running WiPi64 (Debian Trixie, arm64):

```bash
sudo mkdir -p /etc/apt/keyrings
sudo curl -fsSL https://cassianoperin.github.io/wipi64-apt/wipi-archive.asc \
     -o /etc/apt/keyrings/wipi-archive.asc

sudo tee /etc/apt/sources.list.d/wipi.sources >/dev/null <<'EOF'
Types: deb
URIs: https://cassianoperin.github.io/wipi64-apt/
Suites: ./
Components:
Architectures: arm64
Signed-By: /etc/apt/keyrings/wipi-archive.asc
EOF

sudo apt update
```

## Updating WiPi

```bash
sudo apt update
sudo apt install --only-upgrade wipi
```

Check which version is installed and what is available:

```bash
apt policy wipi
```

Install a specific version (the last few releases are kept available):

```bash
sudo apt install wipi=0.23-0
```

## Publishing a new version

See [PUBLISHING.md](PUBLISHING.md).

Short version: attach the `.deb` to a new GitHub Release in this repository.

```bash
gh release create v0.24 wipi_0.24-0_trixie_arm64.deb \
   --repo cassianoperin/wipi64-apt \
   --notes "WiPi64 0.24"
```

A GitHub Actions workflow picks it up automatically, rebuilds and signs the
repository index, and publishes it to GitHub Pages.

## How it works

| Component | Role |
| --- | --- |
| GitHub Releases | Stores the `.deb` files |
| GitHub Actions | Builds `Packages` / `Release`, signs with GPG |
| GitHub Pages | Serves the repository over HTTPS |

The five most recent releases are indexed; older ones stay on GitHub but drop
out of APT. Deleting a release republishes the index without it.

## Signing key

Packages are signed with a dedicated GPG key:

```
2116DBB2E529B5D28E635DA15E671BE26BFDF968
WiPi64 Repo <wipi@cassianoperin>
```
