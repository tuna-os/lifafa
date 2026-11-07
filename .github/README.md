# Lifafa

Lifafa (`org.tunaos.lifafa`) is the email client of the TunaOS office suite,
[tuna-os/gtk-office-suite](https://github.com/tuna-os/gtk-office-suite): a
GTK 4 / libadwaita app for GNOME with native [Tuta](https://tuta.com) and
[Proton Mail](https://proton.me/mail) accounts next to IMAP, POP3 and JMAP.
It is [Hylki](https://github.com/hyprlab/hylki) by hyprlab with those two
kinds of account added; everything else is Hylki's app, merged in daily. It
installs beside upstream Hylki and shares none of its state.

## Install

Lifafa ships as a Flatpak for x86_64 and aarch64. The runtime (GNOME) comes
from [Flathub](https://flathub.org/setup), which has to be set up first.

**From the signed Flatpak repository** (coming with the first release
published there, v1.47.1-tuna6). One command adds the repository and installs
the app:

```sh
flatpak install --user https://tuna-os.github.io/lifafa/org.tunaos.lifafa.flatpakref
```

or add the repository as a remote first:

```sh
flatpak remote-add --user --if-not-exists lifafa \
    https://tuna-os.github.io/lifafa/org.tunaos.lifafa.flatpakrepo
flatpak install --user lifafa org.tunaos.lifafa
```

The repository is signed with `Lifafa (TunaOS) <noreply@tunaos.org>`,
fingerprint `F9D8 8C33 D6A7 AC85 29FE  1FD3 61BC 0585 35DF E2CF`. The public
key is [`fork/flatpak-repo.asc`](../fork/flatpak-repo.asc) and is embedded in
both files above, so Flatpak checks every update against it.

**From a bundle.** Each [release](https://github.com/tuna-os/lifafa/releases/latest)
carries `org.tunaos.lifafa-<version>.flatpak` (x86_64) and
`org.tunaos.lifafa-<version>-aarch64.flatpak`, with their `SHA256SUMS`:

```sh
flatpak install --user org.tunaos.lifafa-<version>.flatpak
```

Bundles of v1.47.1-tuna5 and earlier name no repository and never update.
From v1.47.1-tuna6 a bundle installs the same signed commit as the repository
and updates from it; installing from the repository is still the simpler way.

**Installed a v1.47.1-tuna3, -tuna4 or -tuna5 bundle?** Switch it over to the
repository so it updates. It keeps its app ID, so your accounts, settings and
mail in `~/.var/app/org.tunaos.lifafa` stay as they are:

```sh
flatpak remote-add --user --if-not-exists lifafa \
    https://tuna-os.github.io/lifafa/org.tunaos.lifafa.flatpakrepo
flatpak install --user --reinstall lifafa org.tunaos.lifafa
```

Use `--system` instead of `--user` throughout for a system-wide install.
Coming from upstream Hylki instead? Lifafa starts empty;
`fork/import-from-hylki.sh` copies Hylki's accounts, settings and passwords
over ([Moving over from upstream Hylki](../fork/README.md#moving-over-from-upstream-hylki)).

## Updates

An install from the repository, or from a v1.47.1-tuna6 or later bundle,
follows every release with `flatpak update` and GNOME Software. Releases are
tagged `v<upstream-version>-tuna<N>`; the
[changelog](../fork/CHANGELOG.md) lists the fork's own changes, and each
release's notes name the upstream version it carries. How the repository is
kept, and rolling back with `flatpak update --commit=`, is in
[fork/README.md](../fork/README.md#updates).

## Accounts

- **Tuta:** through a bundled helper built on Tuta's own SDK. Sign in with the
  address and password, plus an authenticator code if the account has one;
  Lifafa keeps the session, never the password. Confidential
  (password-protected) mail to people outside Tuta is supported.
- **Proton Mail, without Bridge:** through a bundled helper built on Proton's
  own libraries, with the authenticator code, mailbox password and CAPTCHA
  Proton asks for. Proton Mail Bridge stays available as its own provider.
- **IMAP and SMTP, POP3 and JMAP:** any provider, as in Hylki.
- **Microsoft 365 and Outlook.com:** browser sign-in, mail through Microsoft
  Graph. **Google:** through GNOME Online Accounts. Any mail account set up in
  GNOME Online Accounts can be imported.

[fork/ACCOUNTS.md](../fork/ACCOUNTS.md) covers Tuta and Proton Mail in detail;
Hylki's [documentation](../docs/DOCUMENTATION.md) covers the rest, including
[OAuth](../docs/DOCUMENTATION.md#oauth-google--microsoft). Security keys (U2F,
WebAuthn) are not supported as a second factor for Tuta or Proton.

## Build from source

Lifafa is built as a Flatpak from a rebranded copy of the tree. It needs
`flatpak-builder`, the GNOME 50 SDK, and the `rust-stable` and `golang` SDK
extensions for 25.08:

```sh
git clone --recurse-submodules https://github.com/tuna-os/lifafa.git
cd lifafa
sh native/tuta-unofficial-sdk/prepare.sh
flatpak-cargo-generator.py Cargo.lock -o cargo-sources.json

rsync -a --delete --exclude=.git --exclude=/target \
    --exclude=/native/tuta-unofficial-sdk/target \
    --exclude=/native/tuta-unofficial-sdk/upstream ./ /tmp/lifafa/
fork/rebrand.sh /tmp/lifafa
flatpak-builder --user --install --force-clean --ccache \
    --state-dir=/tmp/lifafa-state /tmp/lifafa-build \
    /tmp/lifafa/fork/org.tunaos.lifafa.yml
flatpak run org.tunaos.lifafa
```

[fork/README.md](../fork/README.md#building) explains each step and the
release workflow; [fork/HELPERS.md](../fork/HELPERS.md) covers the Tuta and
Proton helpers. A build of the tree as it is, without `rebrand.sh`, is
upstream Hylki (`co.hyprlab.Hylki`), per Hylki's
[BUILDING.md](../docs/BUILDING.md).

## Relationship to upstream

This repository is a soft fork of [hyprlab/hylki](https://github.com/hyprlab/hylki).
Upstream declined native Tuta and Proton accounts
([#584](https://github.com/hyprlab/hylki/pull/584),
[#586](https://github.com/hyprlab/hylki/pull/586)), so the fork carries them
and merges upstream's `main` every day, never rebasing. Everything that is not
Tuta, Proton, the Lifafa branding or fork plumbing goes to upstream first;
[fork/MAINTAINING.md](../fork/MAINTAINING.md) has the rules, and
[fork/README.md](../fork/README.md) how the rebrand works.

Lifafa is not affiliated with or endorsed by hyprlab, Tuta GmbH or Proton AG.
Neither Tuta nor Proton endorses third-party clients built this way; use those
accounts knowing that.

GitHub shows this page in place of the repository's [`README.md`](../README.md),
which stays upstream's own (Hylki's features and documentation) so the daily
merge never conflicts on it.

## Licence

Lifafa is licensed under the GNU Affero General Public License v3.0 or later
([AGPL-3.0-or-later](../LICENSE)), as Hylki is. The Tuta and Proton helpers
are GPL-3.0-or-later and run as separate programs; their licences and the
libraries they link are in [fork/HELPERS.md](../fork/HELPERS.md#licences).

## Issues

Report problems with Lifafa, and anything about Tuta or Proton accounts, on
[tuna-os/lifafa's issues](https://github.com/tuna-os/lifafa/issues). A bug that
also happens in upstream Hylki belongs on
[hyprlab/hylki](https://github.com/hyprlab/hylki/issues); please don't report
Lifafa's Tuta or Proton accounts there.
