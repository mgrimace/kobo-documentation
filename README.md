# Kobo Libra Colour Setup

This is my opinionated documentation for setting up a Kobo Libra Colour (or any Kobo device) for a native, wireless, self-hosted e-book library (i.e., tap a book on the device and it downloads over WiFi) with Hardcover progress syncing, plus streaming manga with no conversion needed.

> [!NOTE]
> I've now added a new CrossPoint 'micro' reader for work/travel/fun, the EegoRead A4. To sync between my Kobo and a Crosspoint device, I replaced Calibre-Web-Automated (Next Gen) with Bookorbit. I've updated the instructions to set up progress sync between that device and my Kobo as the main reader. This should work for any CrossPoint device. 

The organizational philosophy here is to keep the book and manga libraries completely separate:

- The native Kobo interface is used exclusively for reading books.
- KoReader is used exclusively as the entry point for the manga library. From the Kobo home screen, NickelMenu provides a quick launch directly into the Rakuyomi manga library, using KoReader and a few supporting plugins. This keeps the manga collection and reading experience completely separate from the native Kobo book library.
- The central library service is [BookOrbit](https://github.com/bookorbit/bookorbit), which serves as both my digital library and hub to sync reading progress between all my devices and back to Hardcover.app.

## Contents

- [`patches/kobopatch.yaml`](patches/kobopatch.yaml) — kobopatch overrides used on this device
- [`nicklemenu/config`](nicklemenu/config) — NickelMenu configuration
- [`books/compose.yaml`](books/compose.yaml) / [`books/example.env`](books/example.env) — Calibre-Web-Automated (Next-Gen) Docker Compose stack

## 1. Prepare the device

### Patch Nickel with kobopatch

Use the latest patches from the [MobileRead Kobo Developer forum](https://www.mobileread.com/forums/forumdisplay.php?f=247).

My [`patches/kobopatch.yaml`](patches/kobopatch.yaml) contains my personal preferences, for example:

- Removing the third row (footer) and increasing cover size on the home screen
- Increasing cover size on the library and series pages

> [!CAUTION]
> Make sure the version and source (`in:`) match your exact firmware!

Copy/merge the `overrides:` section from [`patches/kobopatch.yaml`](patches/kobopatch.yaml) into your own kobopatch config, run kobopatch, and install the resulting `KoboRoot.tgz` on the device.

### Add NickelMenu

Install [NickelMenu](https://github.com/pgaskin/NickelMenu) to add custom menu entries (toggles, quick actions, launching KoReader, etc.) to the native Kobo UI.

My config lives in [`nicklemenu/config`](nicklemenu/config) and adds a custom icon and entries like dark mode / invert screen quick toggles, manually trigger a USB connection (useful for setup without unplugging the cable), and quick access to reading stats, reboot, and shutdown. These options are available from the home screen, reader, library, and browser menus.

### Add KoReader

Install [KoReader](https://github.com/koreader/koreader). It's used solely for reading manga through Rakuyomi (see [Section 3](#3-setup-manga)).

### Optional device tweaks

- [NickelTypeFix](https://github.com/nicoverbruggen/NickelTypeFix) — improves native typesetting/justification
- [NickelDissolve](https://github.com/nicoverbruggen/NickelDissolve) — animates page turns
- [NickelSync](https://github.com/wolffshots/nickelsync) - trigger a sync from within a book, useful for quickly pulling/pushing progress

## 2. Setup the book library

> [!TIP]
> See the migration instructions below if you started on Calibre-Web-Automated (Next-Gen).

### Self-host BookOrbit

Self-host [BookOrbit](https://github.com/bookorbit/bookorbit). I deployed it in a Proxmox LXC using the [Community Helper Script](https://community-scripts.org/scripts/bookorbit). You can also install it via Docker from their repository.

### Basic BookOrbit configuration

In the BookOrbit web UI:

1. Connect your Hardcover account.
2. Add a Kobo device and note the generated Kobo API address. You'll need it on the device in the next step.
3. Configure your library and review the book matches before importing reading history.

### Migrating from Calibre-Web-Automated (Next-Gen)

I migrated the library files from CWA-NG, but deliberately did **not** migrate its reading history or progress data. Instead, I use [Hardcover.app](hardcover.app) as the source of truth for reading status and history.

1. Copy the raw Calibre Library folder from CWA-NG into BookOrbit's books folder.
2. Let BookOrbit scan the library and confirm the books have been imported.
3. Before importing reading history, remove books you no longer want in the library. Do this first, since reading-history entries can remain even if you later remove a book.
4. Tidy up Hardcover before importing anything:
   - Confirm read / unread and want-to-read statuses.
   - Review reading start and finish dates.
   - Remove incorrect or duplicate reading-history entries.
5. Pull the Hardcover data into BookOrbit.

I found this much cleaner than importing CWA-NG reading data directly. My previous CWA-NG import introduced conflicting dates and duplicate reads. Cleaning up Hardcover first and using it as the source of truth avoids carrying those problems into BookOrbit.

Matching generally works well when the book already exists in Hardcover, but review unmatched or incorrect matches. Metadata cleanup is optional. I prefer to tidy important matches manually, or configure the automatic matching parameters before running a bulk match.

### Setup Pangolin

> [!WARNING]
> If you use Pangolin, you'll need to bypass its authentication for the following integration endpoints. This allows device and app requests through the Access Gate; BookOrbit still performs its own authentication.

Add the following paths as **Bypass Auth** rules on the BookOrbit resource:

| Path | Purpose |
|---|---|
| `/api/v1/opds*` | OPDS catalog and downloads |
| `/api/v1/opds` | no wildcard, OPDS catalog and downloads |
| `/api/v1/koreader*` | KOReader / KOSync |
| `/api/v1/kobo/*` | Kobo sync |
| `/api/v3/*` | Kobo annotations and highlights |
| `/api/UserStorage/*` | Kobo device storage |
| `/api/v1/auth/*` | Login/authorization |
| `/api/v1/books/*` | Also used for sync |
| `/api/v1/reading-progress*` | CrossPoint Sync progress reads and writes |

The last path is particularly important for the CrossPoint Sync BookOrbit connector. The connector uses BookOrbit's newer reading-progress API for both reading and writing progress. Without this bypass, Pangolin's authentication can interfere with synchronization.

#### BookOrbit iOS app

For the iOS app, create a **Shareable Link** from Pangolin's main navigation and associate it with the BookOrbit resource.

In the app's custom HTTP headers, add the Shareable Link's Token ID and Token:

| Header | Value |
|---|---|
| `P-Access-Token-Id` | Shareable Link Token ID |
| `P-Access-Token` | Shareable Link Token |

The Pangolin token gets the request through the Access Gate; BookOrbit still performs its own user authentication. Treat the token like a password.

### Point the Kobo at your self-hosted library

Mount the Kobo over USB and edit:

```text
.kobo/Kobo/Kobo eReader.conf
```

Replace the `api_endpoint` value with the generated Kobo API address from BookOrbit's Kobo device configuration.

After unmounting and reconnecting to WiFi, the Kobo will sync against BookOrbit. Books can then be tapped to download directly on-device.

> [!TIP]
> Large libraries often take multiple syncs the first time. If you have errors, force a full resync in BookOrbit's settings.

## 3. Setup manga

All manga reading happens inside KoReader via the Rakuyomi plugin, kept fully separate from the native book library.

- Install the [Rakuyomi](https://github.com/tachibana-shin/rakuyomi) KoReader plugin, and use the **Kindle** release build.
- Install [stretch.koplugin](https://github.com/mgrimace/stretch.koplugin) so manga pages fill the available screen without extra scrolling.
- Install [startrakuyomi.koplugin](https://github.com/mgrimace/startrakuyomi.koplugin) so launching KoReader drops you straight into your Rakuyomi library.

### Optional KoReader plugins

- [simpleui.koplugin](https://github.com/doctorhetfield-cmd/simpleui.koplugin) — simple menu bar actions similar to tapping the top of the screen in the native Kobo UI (quit back to Nickel, brightness, etc.)
- [appstore.koplugin](https://github.com/omer-faruq/appstore.koplugin) — easier plugin discovery, installation, and updates

### Manga setup

Launch KoReader from the NickelMenu entry, which should bring you right into Rakuyomi's library view. Add your manga sources, and add titles into Rakuyomi's library. Quit to return to Kobo to read.

## 4. Setup CrossPoint device

The EEGO A4 is my secondary reader for work, travel, and general use. It runs CrossPoint firmware and uses BookOrbit as its library and progress-sync hub.

### Self-host CrossPoint Sync

I self-host [CrossPoint Sync](https://github.com/crosspoint-reader/crosspoint-sync) separately from the BookOrbit LXC, on my `books` host.

Deploy it using the [CrossPoint Sync Docker image](https://github.com/crosspoint-reader/crosspoint-sync). Configure its persistent data storage and expose the service through Pangolin if you need remote access.

In CrossPoint Sync, add BookOrbit as a connector using:

- **Server:** your BookOrbit URL
- **Username:** your regular BookOrbit account username
- **Password:** your regular BookOrbit account password

This is the native BookOrbit connector, not the KOReader plugin or the OPDS account. Complete the connector setup and verify that it can connect to BookOrbit.

### Install CrossPoint

Install [CrossPoint](https://crosspointreader.com/install) on the EEGO A4 using the [official web installer](https://crosspointreader.com/install).

Connect the reader over USB, select the EEGO A4 in the installer, and follow the prompts to flash the firmware. Complete the initial setup and connect the reader to WiFi.

### Connect the reader to CrossPoint Sync

On the reader, configure CrossPoint Sync with:

- **Server type:** CrossPoint
- **Server URL:** your self-hosted CrossPoint Sync URL
- **Credentials:** your CrossPoint Sync account

Save the settings and run a manual sync to verify the connection. I also mapped a long press of the Home button to trigger a sync.

### Add BookOrbit as an OPDS catalog

OPDS is separate from CrossPoint Sync. It lets the reader browse and download books from BookOrbit.

In BookOrbit, enable OPDS and create an account with the `opds_access` permission.

On the EEGO A4, add an OPDS server using:

```text
https://<bookorbit-host>/api/v1/opds
```

Enter the OPDS account credentials and save. The catalog should then be available from the reader for browsing and downloading books.

### Progress sync

BookOrbit acts as the central hub for reading progress across both devices and for updates to Hardcover. The goal is to read on one device, sync, then pick up on the other device where I left off.

For example:
1. Read on the Kobo or EEGO A4.
2. Trigger a sync on that device.
3. Pick up the other reader and trigger a sync.
4. Continue reading from the synchronized position.

The sync paths are:

- **Kobo Libra Colour:** Uses BookOrbit's native Kobo sync API. Sync from the Kobo home screen.
- **EEGO A4:** Uses CrossPoint Sync, with BookOrbit configured as its connector. Sync by long-pressing the Home button.

BookOrbit receives the progress updates and coordinates synchronization between the devices and Hardcover.