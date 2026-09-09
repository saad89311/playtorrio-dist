# Maintainer notes — PlayTorrio Ad Hoc distribution

**KEEP THIS FILE LOCAL. Do not commit it to the public repo.**

All identifiers below are placeholders. Fill them in on your own machine, or
set them as shell variables each time instead of writing them down.

---

## One-time setup on a new Mac

Set these once per terminal session:

```bash
export SIGN_ID="Apple Distribution: <YOUR NAME> (<TEAM_ID>)"
export WORKDIR="$HOME/Downloads/work"
export PROFILE="$HOME/Downloads/<ProfileName>.mobileprovision"
export OUT="$HOME/Downloads/PlayTorrio-signed.ipa"
```

To see the exact identity string without hardcoding it anywhere:

```bash
security find-identity -v -p codesigning
```

---

## Adding a device

1. Apple Developer portal → **Devices** → **+** → register the UDID
2. **Profiles** → the Ad Hoc profile → **Edit** → tick every device → **Save** → **Download**

Tick all devices at once rather than repeating this per person.
The membership allows 100 iOS devices per year; slots reset on the renewal date only.

---

## Re-signing

Assumes the unpacked app is in `$WORKDIR` and the fresh profile is at `$PROFILE`.

```bash
cd "$WORKDIR"
APP=$(ls -d Payload/*.app)

cp "$PROFILE" "$APP/embedded.mobileprovision"
security cms -D -i "$APP/embedded.mobileprovision" > profile.plist
/usr/libexec/PlistBuddy -x -c 'Print :Entitlements' profile.plist > ent.plist

for f in "$APP"/Frameworks/*.framework; do
  rm -rf "$f/_CodeSignature"
  codesign -f -s "$SIGN_ID" --timestamp=none "$f"
done

codesign -f -s "$SIGN_ID" --timestamp=none --entitlements ent.plist "$APP"
codesign --verify --deep --strict "$APP" && echo "signature OK"

rm -f "$OUT"
zip -qry "$OUT" Payload
```

Notes:

- Frameworks must be signed **before** the main app, never after.
- `zip -y` preserves the symlinks inside the frameworks. Without it, the build will not install.
- zsh aborts a whole command block if a glob matches nothing. Do not add loops
  over patterns that may have no matches (e.g. `*.dylib` when there are none).
- `profile.plist` and `ent.plist` are generated files. They are safe to delete
  but should never be committed anywhere.

---

## Publishing

1. GitHub → **Releases** → **Draft a new release** → new tag (e.g. `v1.1.4-2`)
   → attach the `.ipa` → publish
2. Edit `manifest.plist` in the repo — update the tag inside the asset URL
3. Update the version number shown in `index.html` if it changed

Verify both endpoints return `200`:

```bash
curl -sI https://<user>.github.io/<repo>/manifest.plist | head -1
curl -sIL https://github.com/<user>/<repo>/releases/download/<tag>/PlayTorrio-signed.ipa \
  | grep -E "^HTTP|content-length"
```

Both URLs must be HTTPS. `itms-services://` fails silently over plain HTTP.

---

## Privacy consideration

A public repo means anyone can download the `.ipa` and read
`Payload/*.app/embedded.mobileprovision` inside it. That file contains:

- the team identifier and team name
- the certificate
- **the UDID of every device in the profile**

If that matters, host the build somewhere access-controlled instead of a public
GitHub release — a link service with expiring URLs, or a private server.

---

## Expiry

| Thing | Lifetime | What happens when it ends |
|---|---|---|
| Ad Hoc profile | 1 year from creation | Installed apps stop launching |
| Distribution certificate | 1 year, renewable | Same, plus re-signing fails |
| Developer membership | Annual | Certificate is revoked; all builds die |

Diary the profile expiry date. Re-signing before it lapses is a five-minute job;
finding out from a tester that the app stopped opening is not.
