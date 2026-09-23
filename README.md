# Kobo Libra Colour Setup

Opinionated documentation for setting up a Kobo Libra Colour (or any Kobo device) for a native, wireless, self-hosted e-book library (i.e., tap a book on the device and it downloads over WiFi) with Hardcover syncing, plus streaming manga with no conversion needed.

My organizational philosophy here is to keep books and manga libraries separate and tidy:
- The **native Kobo interface** is used for reading books (synced wirelessly from a self-hosted library).
- **KoReader** is used exclusively as an entry point into a manga library, so the two collections and reading experiences never mix.

## Contents

- [`patches/kobopatch.yaml`](patches/kobopatch.yaml) — kobopatch overrides used on this device
- [`nicklemenu/config`](nicklemenu/config) — NickelMenu configuration
- [`books/compose.yaml`](books/compose.yaml) / [`books/example.env`](books/example.env) — Calibre-Web-Automated (Next-Gen) Docker Compose stack

## 1. Prepare the device

### Patch Nickel with kobopatch

Use the latest patches from the [MobileRead kobopatch forum](https://www.mobileread.com/forums/forumdisplay.php?f=247).

My [`patches/kobopatch.yaml`](patches/kobopatch.yaml) contains my personal preferences layered on top of the standard patch set, for example:
- Removing the third row (footer) and increasing cover size on the home screen
- Increasing cover size on the library and series pages

Copy/merge the `overrides:` section from [`patches/kobopatch.yaml`](patches/kobopatch.yaml) into your own kobopatch config, run kobopatch, and install the resulting `KoboRoot.tgz` on the device.

### Add NickelMenu

Install [NickelMenu](https://github.com/pgaskin/NickelMenu) to add custom menu entries (toggles, quick actions, launching KoReader, etc.) to the native Kobo UI.

My config lives in [`nicklemenu/config`](nicklemenu/config) and adds things like dark mode / invert screen quick toggles, manually trigger a USB connection (useful for setup without unplugging cable),and quick access to reading stats, reboot, and shutdown. These options are available from the home screen, reader, library, and browser menus.

### Add KoReader

Install [KoReader](https://github.com/koreader/koreader). It's used solely for reading manga through Rakuyomi (see [Section 3](#3-setup-manga)) — not for regular e-books, which stay in the native Kobo library.

### Optional device tweaks

- [NickelTypeFix](https://github.com/nicoverbruggen/NickelTypeFix) — improves native typesetting/justification
- [NickelDissolve](https://github.com/nicoverbruggen/NickelDissolve) — animates page turns

## 2. Setup the book library

### Self-host Calibre-Web-Automated (Next-Gen)

Self-host [Calibre-Web-NextGen](https://github.com/new-usemame/Calibre-Web-NextGen) (CWA-NG). Next-Gen is the actively maintained fork of the original Automated project and includes UI and feature improvements.

My Compose stack is in [`books/compose.yaml`](books/compose.yaml), with environment variable placeholders in [`books/example.env`](books/example.env). 

> [!NOTE]
> My library is located on a network drive and requires `NETWORK_SHARE_MODE: true`. If yours is not, remove that.

Copy the example env file, fill in your Hardcover token, and bring the stack up:

```bash
cp books/example.env books/.env
docker compose -f books/compose.yaml up -d
```

### Basic CWA-NG configuration

In the CWA-NG web UI, under your account settings:

- Enable the **Kobo sync API** and note the generated API address, you'll need it on the device in the next step.
- Set **"stretch to fill"** as the default cover thumbnail profile. This mildly stretches cover art to fill the entire screen, which looks better on the full-colour Libra Colour display than the black bars you get from letterboxed covers.

### Point the Kobo at your self-hosted library

Mount the Kobo over USB and edit:

```
.kobo/Kobo/Kobo eReader.conf
```

Replace the `api_endpoint` value with your self-hosted Kobo sync API address from CWA-NG. After unmounting and reconnecting to WiFi, your library will sync natively and books can be tapped to download directly on-device.

> [!TIP]
> Large libraries often take multiple syncs the first time. If you have errors force a full resync in the CWA-NG settings.

## 3. Setup manga

All manga reading happens inside KoReader via the Rakuyomi plugin, kept fully separate from the native book library.

- Install the [Rakuyomi](https://github.com/tachibana-shin/rakuyomi) KoReader plugin, and use the **Kindle** release build.
- Install [stretch.koplugin](https://github.com/mgrimace/stretch.koplugin) so manga pages fill the available screen without extra scrolling.
- Install [startrakuyomi.koplugin](https://github.com/mgrimace/startrakuyomi.koplugin) so launching KoReader drops you straight into your Rakuyomi library.

### Optional manga plugins

- [simpleui.koplugin](https://github.com/doctorhetfield-cmd/simpleui.koplugin) — simple menu bar actions similar to tapping the top of the screen in the native Kobo UI (quit back to Nickel, brightness, etc.)
- [appstore.koplugin](https://github.com/omer-faruq/appstore.koplugin) — easier plugin discovery, installation, and updates

### Manga setup

Launch KoReader from the NickelMenu entry, which should bring your right into Rakuyomi's library view. Add your manga sources, and add titles into Rakuyomi's library. Quit to return to kobo to read.
