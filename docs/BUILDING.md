# Building from source

Hylki needs the Rust toolchain ([rustup](https://rustup.rs)), a C compiler,
and the GTK 4 / libadwaita / WebKitGTK 6 development libraries, plus
OpenSSL, gettext and Wayland headers. At runtime it needs a Secret Service
provider (e.g. gnome-keyring).

## Dependencies

**Fedora**

```sh
sudo dnf install gcc gcc-c++ gtk4-devel libadwaita-devel webkitgtk6.0-devel \
    poppler-glib-devel openssl-devel gettext-devel wayland-devel
```

**Debian / Ubuntu**

```sh
sudo apt install build-essential pkg-config libgtk-4-dev libadwaita-1-dev \
    libwebkitgtk-6.0-dev libpoppler-glib-dev libssl-dev gettext libwayland-dev
```

## Build and install

```sh
git clone https://github.com/hyprlab/hylki.git
cd hylki
cargo build --release
./install.sh          # installs the binary, icon and .desktop file into ~/.local
./uninstall.sh        # removes them again (--purge also removes settings and the mail cache)
```

## Tuta helper

Tuta accounts go through `hylki-tuta`, a separate program (GPL-3.0-or-later):
`tuta-helper` from [tuta-unofficial-sdk](https://github.com/tuna-os/tuta-unofficial-sdk),
the `native/tuta-unofficial-sdk` submodule, installed under the name
`hylki-tuta`. Hylki looks for it next to the `hylki` executable first, then
on `PATH`, so every package installs it in the same `bin` directory as
`hylki`. Started as `hylki-tuta` it names its sessions "Hylki" and reads
`HYLKI_TUTA_DEBUG`. Without it, everything but Tuta accounts works.

The helper builds against the Tuta SDK, which is generated from the
submodules inside the SDK repository (tutabridge, and the tutanota repository
inside that) plus the SDK repository's own patches. Check the submodules out
and prepare the SDK once, and again whenever the submodule moves:

```sh
git submodule update --init --recursive
sh native/tuta-unofficial-sdk/prepare.sh
```

The script needs `git` and `python3`, no network. It runs tutabridge's
`sdk-prepare.py`, which verifies the generated tree against the digests the
submodule pins, then writes that tree with the patches applied, beside
tutabridge's `tutabridge-tuta` crate, to `native/tuta-unofficial-sdk/prepared`
(untracked), which the helper builds from. On a repeat run it only checks.
Then build the helper and put it beside `hylki` under the name the app looks
for:

```sh
(cd native/tuta-unofficial-sdk && cargo build --release --locked --bin tuta-helper)
install -Dm755 native/tuta-unofficial-sdk/target/release/tuta-helper ~/.local/bin/hylki-tuta
```

The SDK repository has its own `Cargo.lock`, tests and CI; its README
describes the protocol and lists its SDK patches. Moving to a new release is
a submodule bump to its tag. `.github/workflows/tuta-helper.yml` checks that
the submodule sits on a release tag and that the helper builds and answers
as `hylki-tuta`.

## Proton helper

Proton Mail accounts that sign in natively (rather than through Proton Mail
Bridge) go through `hylki-proton`, a separate Go program (GPL-3.0-or-later):
`proton-helper` from [proton-unofficial-sdk](https://github.com/tuna-os/proton-unofficial-sdk),
the `native/proton-unofficial-sdk` submodule, installed under the name
`hylki-proton`. It behaves the same under either name. It is found the same
way as the Tuta helper: next to `hylki`, then on `PATH`. Without it,
everything but native Proton accounts works.

It needs the submodule checked out (`git submodule update --init`), Go
1.26.1 or newer (the `go` line in `native/proton-unofficial-sdk/go.mod`) and
nothing from the network: its dependencies are committed in the SDK's
`vendor/`.

```sh
(cd native/proton-unofficial-sdk && go build -mod=vendor -trimpath -o hylki-proton ./cmd/proton-helper)
install -Dm755 native/proton-unofficial-sdk/hylki-proton ~/.local/bin/hylki-proton
```

The SDK repository has its own tests, which run offline against
go-proton-api's in-process server, and its own CI; dependency bumps and
`go mod vendor` happen there. Its README describes the JSON it speaks.
Moving to a new release is a submodule bump to its tag.
`.github/workflows/proton-helper.yml` checks that the submodule sits on a
release tag, that the helper builds offline, and that run as `hylki-proton`
it answers every operation the app sends with the envelope the app parses.

## Translations

`tools/build-locale.sh` compiles `po/*.po` so a source-tree run picks them up;
the Flatpak, the RPM, the Debian package and `install.sh` all do it as part of
their own build.
See [po/README.md](../po/README.md).

## Flatpak

The manifest is `co.hyprlab.Hylki.yml`, built with `flatpak-builder` (or
`org.flatpak.Builder`). Its Rust dependencies come from `cargo-sources.json`,
which is not committed; regenerate it whenever `Cargo.lock` changes:

```sh
python3 flatpak-cargo-generator.py Cargo.lock -o cargo-sources.json
```

The Tuta helper's come from `native/tuta-unofficial-sdk/cargo-sources.json`,
which the SDK repository commits (its `scripts/cargo-sources.sh` writes it,
and its CI fails when it is stale).

The `hylki-tuta` module builds `tuta-helper` into `/app/bin` beside `hylki`,
with `hylki-tuta` a symlink to it,
and the `hylki-proton` module builds `proton-helper` there, with
`hylki-proton` a symlink to it, using the
`org.freedesktop.Sdk.Extension.golang//25.08` SDK extension (install it
beside the Rust one), offline from `native/proton-unofficial-sdk/vendor`.
Check the submodules out and prepare the Tuta SDK first, as above: neither
the submodules nor the generated tree are in the pinned git source, so
the manifest takes them from the checkout it is built in. To build the working
tree rather than the tagged release, write a local manifest:

```sh
tools/local-manifest.sh /tmp/hylki-local.yml
flatpak-builder --user --install --force-clean build-dir /tmp/hylki-local.yml
```

## RPM and Debian packages

`tools/build-packages.sh rpm` and `tools/build-packages.sh deb` produce
`packaging/out/hylki-<version>-1.fc44.x86_64.rpm` and
`packaging/out/hylki_<version>-1_amd64.deb`; with no argument it builds both.
A beta's version is written `1.45.0~beta.1` in both, which sorts before
`1.45.0`. Each is built for the machine it runs on: the ARM64 ones
(`.aarch64.rpm`, `_arm64.deb`) come from `.github/workflows/build-arm64.yml`,
which builds them on GitHub's arm64 runners and installs each in a clean
Fedora 44 or Debian 13 container to check it.

The RPM wraps a release build from the host, so it is built on Fedora. The
Debian package is built inside a Debian 13 container (the script starts one
with podman), because a binary built on Fedora needs a newer C library than
Debian has, and Debian 13 is the oldest release with libadwaita 1.7. On a
Debian host with the dependencies above and `dpkg-dev`, it builds in place.
`dpkg-shlibdeps` works out the package's dependencies from the binaries.
Both packages carry the Tuta helper as `/usr/bin/hylki-tuta`; the script
prepares the Tuta SDK and builds the helper itself, so the submodules have
to be checked out. They carry the Proton helper as `/usr/bin/hylki-proton`
too, so the RPM build needs `go` on the host; the Debian container installs
Debian's, and a Go older than `native/proton-unofficial-sdk/go.mod` asks for
fetches the right one (`GOTOOLCHAIN=auto`).

## AppImage

`tools/build-appimage.sh` produces `packaging/out/Hylki-<arch>.AppImage`, or
`Hylki-Beta-<arch>.AppImage` when the version is a beta, which is then built
with the `beta` feature as the Flatpak beta is (#235). It runs inside an Arch
container (the script starts one with podman; in CI it is already in one),
because the bundle carries every library down to the C library and those have
to be current. The library collecting is [pkgforge's
`quick-sharun`](https://github.com/pkgforge-dev/Anylinux-AppImages), fetched
by the script rather than vendored. `HYLKI_APPIMAGE_SRC` names another
checkout to build with this script, such as an older tag.

The bundle also carries the spell-check dictionaries and a few fonts with a
fontconfig file of their own (`packaging/appimage/fonts-fallback.conf`), which
the message view falls back to when the system's fonts hang it (#296), and the
Tuta and Proton helpers beside `hylki`, built the same way as for the packages.

`.github/workflows/build-appimage.yml` builds it for x86_64 and aarch64 on
every tag. Nothing is published from CI: the release attaches the two
AppImages, and Flatout on hylki.hyprlab.co serves each channel's newest one
with the `.zsync` file AppImage updaters read.

## Your own OAuth client

Google and Dropbox clients can be compiled in at build time; see
[Configuration](DOCUMENTATION.md#oauth-google--microsoft).
