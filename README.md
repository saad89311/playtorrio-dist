# PlayTorrio — iOS Test Builds

Ad Hoc distribution of the PlayTorrio iOS app for registered test devices.

**Current version:** 1.1.4 (build 15)
**Requires:** iPhone running iOS 14.0 or later
**Install page:** https://saad89311.github.io/playtorrio-dist/

---

## For testers — how to install

> **Your device must be registered first.** This build only installs on iPhones whose UDID was added to the signing profile. If you have not sent your UDID to the developer, the download will start and then fail. See [Getting your UDID](#getting-your-udid) below.

1. Open **https://saad89311.github.io/playtorrio-dist/** in **Safari**.

   If the link arrived over WhatsApp, Telegram or another app, tap the share icon and choose **Open in Safari** first. The install button does not work inside in-app browsers.

2. Tap **Install**, then confirm on the popup.

3. Wait for the app icon to finish downloading on your Home Screen. Stay on Wi-Fi or mobile data — the app checks in with Apple on first launch.

4. Go to **Settings → General → VPN & Device Management**.

5. Under *Developer App*, tap **Muhammad Saad Khan**, then tap **Trust** and confirm.

6. Open the app from the Home Screen.

---

## Getting your UDID

The UDID is a unique identifier for your iPhone. It is not your serial number and not your phone number.

**With a Mac:**

1. Connect the iPhone by cable, unlock it, tap **Trust** if asked.
2. Open **Finder** — the iPhone appears in the left sidebar under *Locations*.
3. Click it. Under the device name there is a grey line showing the model and storage.
4. **Click that line repeatedly** — it cycles through model, serial number, then **Identifier (UDID)**.
5. Right-click → **Copy**, and send it to the developer.

The UDID looks like `00008020-000C1D311441002E`.

---

## Troubleshooting

| What you see | What it means |
|---|---|
| Nothing happens when you tap Install | You are not in Safari. Open the page in Safari and try again. |
| "Unable to install" | Your device is not in the signing profile. Send your UDID to the developer. |
| Icon appears greyed out and stops | The download was interrupted. Delete the icon, then tap Install again. |
| App installs but will not open | You skipped step 5. Go to Settings and trust the developer. |
| "Untrusted Enterprise Developer" | Same as above — trust the developer in Settings. |
| App stops working after some time | The signing profile expired. Ask the developer for a new build. |

---

## For the maintainer — publishing a new build

### Adding a device

1. **[developer.apple.com](https://developer.apple.com/account/resources/devices/list) → Devices → +** — register each new UDID.
2. **Profiles → PlayTorrio AdHoc → Edit** — tick every device that should be included → **Save** → **Download**.

Tick all devices at once rather than repeating this per person. The account allows 100 devices per membership year.

### Re-signing

Requires the unpacked app in `~/Downloads/work/` and the downloaded profile in `~/Downloads/`.

```bash
cd ~/Downloads/work
APP=$(ls -d Payload/*.app)
IDENTITY="Apple Distribution: Muhammad Saad Khan (858793HM6U)"

cp ~/Downloads/PlayTorrio_AdHoc.mobileprovision "$APP/embedded.mobileprovision"
security cms -D -i "$APP/embedded.mobileprovision" > profile.plist
/usr/libexec/PlistBuddy -x -c 'Print :Entitlements' profile.plist > ent.plist

for f in "$APP"/Frameworks/*.framework; do
  rm -rf "$f/_CodeSignature"
  codesign -f -s "$IDENTITY" --timestamp=none "$f"
done

codesign -f -s "$IDENTITY" --timestamp=none --entitlements ent.plist "$APP"
codesign --verify --deep --strict "$APP" && echo "signature OK"

rm -f ~/Downloads/PlayTorrio-signed.ipa
zip -qry ~/Downloads/PlayTorrio-signed.ipa Payload
```

Note: `zip -y` preserves the symlinks inside the frameworks. Without it the build will not install.

### Publishing

1. **Releases → Draft a new release** — new tag (e.g. `v1.1.4-2`), attach `PlayTorrio-signed.ipa`, publish.
2. Edit `manifest.plist` in this repo and update the tag in the asset URL to match.
3. Update the version shown in `index.html` if it changed.

### Verifying

```bash
curl -sI https://saad89311.github.io/playtorrio-dist/manifest.plist | head -1
curl -sIL https://github.com/saad89311/playtorrio-dist/releases/download/v1.1.4/PlayTorrio-signed.ipa | grep -E "^HTTP|content-length"
```

Both must return `200`. Both URLs must be HTTPS — `itms-services` fails silently over plain HTTP.

---

## Repository contents

| File | Purpose |
|---|---|
| `index.html` | Install landing page served by GitHub Pages |
| `manifest.plist` | iOS install manifest pointing at the release asset |
| `README.md` | This file |

The `.ipa` itself is attached to a [Release](../../releases), not committed to the repository.

---

## Notes

- This is an **Ad Hoc** build, not an App Store or TestFlight release. It is intended for named test devices only.
- The signing profile expires **9 September 2027**. Builds stop launching after that and must be re-signed.
- If the distribution certificate is revoked or the developer membership lapses, already-installed copies will stop opening.
