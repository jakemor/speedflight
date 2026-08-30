---
name: speedflight
description: Share a local iOS build with a registered iPhone that is not connected to this Mac. Cuts a cloud-signed build, uploads it to Speedflight (speedflight.dev), and posts a link in the chat that installs from Safari. Use it proactively, without being asked, whenever the user wants a build on their device and the device is not plugged in or reachable (run-on-device fails, "send me the build", "I'm away from my desk", "put this on my phone"). Also use on "/speedflight" or "share a build", and to set up Speedflight for a repo.
user-invocable: true
---

# Speedflight

Base URL: `https://speedflight.dev`

Speedflight stores signed iOS builds and serves one page per app. Every build
on the page shows a title, release notes (what changed, what to test), the
version and build number, when it was cut, the branch, commit, and author.
A Get button installs it from Safari on a registered iPhone. A link downloads
the IPA.

You run everything on the user's Mac. There is no account and no login.

The point of this skill is one thing: get a local build onto a registered
device when that device is not next to the Mac. When a run-on-device step
fails because the phone is unreachable, or the user asks for a build while
away, do this without being asked. Every run ends with the page link posted
in the chat.

## Security model

Two identifiers, two jobs:

- `SPEEDFLIGHT_SECRET` is the upload key. You mint it once per repo and keep
  it in `.env.speedflight` (gitignored). It is the user's, for this app. Do
  not share it, print it, or commit it.
- The page id is derived on the server: `sha256("<bundleId>\n<secret>")`,
  first 32 hex chars. That is what the share URL contains. Whoever has the
  link can install and download. They cannot upload.

The IPA is signed by Apple with an ad hoc profile, so it installs only on
devices registered in the signing Apple account. The link is the auth; the
signature is the safety.

## Modes

- First run in a repo (no `.env.speedflight`, or no `scripts/speedflight.sh`):
  do **Setup**, then **Share a build**.
- Every later run: **Share a build**.
- If the user asks for it to run without a Mac, or from a Linux devbox or
  sandbox: **On request: run it in GitHub Actions**.

---

# Setup (once per repo)

## 1. Discover the project

Find, and confirm with the user in one short table:

| Fact | How |
|---|---|
| `PROJECT` | `ls *.xcworkspace *.xcodeproj` (also under `ios/`). Prefer a workspace if one exists. If `project.yml` exists, the project is XcodeGen-generated and must be regenerated with `xcodegen generate` before every build. |
| `SCHEME` | `xcodebuild -list -json -project "$PROJECT"`. Pick the app scheme. |
| `BUNDLE_ID` | `rg -n PRODUCT_BUNDLE_IDENTIFIER` in the pbxproj or `project.yml`. Do not use a `.dev` variant unless the user says so. |
| `TEAM_ID` | `rg -n DEVELOPMENT_TEAM` in the pbxproj or `project.yml`. |
| Deep link | See step 3. |

## 2. App Store Connect key

Cloud signing needs an App Store Connect API key with role **Admin** or
**App Manager**, belonging to `TEAM_ID`. Look for one before asking:

```bash
ls ~/private_keys/AuthKey_*.p8 2>/dev/null
asc profiles list 2>/dev/null
cat .env.speedflight 2>/dev/null
```

If none exists, tell the user to create one at App Store Connect → Users and
Access → Integrations → App Store Connect API → Team Keys → Generate API Key.
The `.p8` downloads once; save it as `~/private_keys/AuthKey_<KEY_ID>.p8`.

Cloud signing only. Never revoke, delete, or create certificates or profiles
by hand, never open Keychain Access, never export a `.p12`, and never switch
a target to manual signing. `-allowProvisioningUpdates` with the key flags
does all of it.

## 3. Deep link

The install page opens the app after install with a URL scheme. Check for
one:

```bash
rg -n "CFBundleURLSchemes" -A3 --glob '*.plist' --glob 'project.yml' --glob '*.pbxproj'
```

If the app has none, add one. Pick a short lowercase scheme from the app name
(for example `tressa://`), confirm it with the user, and add it:

- XcodeGen (`project.yml`), under the app target:
  ```yaml
  info:
    properties:
      CFBundleURLTypes:
        - CFBundleURLName: com.example.app
          CFBundleURLSchemes: [tressa]
  ```
- Plain Xcode: add `CFBundleURLTypes` to the target's `Info.plist`.

Commit that change. The deep link goes in `.env.speedflight` as
`SPEEDFLIGHT_DEEP_LINK=tressa://`.

## 4. Author

The build page shows who cut the build. Resolve the GitHub username once:

```bash
gh api user --jq .login 2>/dev/null || git config --get github.user
```

If neither answers, ask the user. Store it as `SPEEDFLIGHT_AUTHOR`.

## 5. Icon (optional)

If the repo has an asset catalog icon, the page shows it. Find the 1024 PNG:

```bash
find . -path '*.appiconset/*.png' -not -path '*/node_modules/*' | head
```

Store the path as `SPEEDFLIGHT_ICON`. Skip this if there is no plain PNG.

## 6. Write `.env.speedflight`

Mint the secret and write the file at the repo root (or `ios/` if the app
lives there):

```bash
cat > .env.speedflight <<EOF
ASC_KEY_ID=ABC123DEFG
ASC_ISSUER_ID=12345678-abcd-....
SPEEDFLIGHT_SECRET=$(openssl rand -hex 24)
SPEEDFLIGHT_DEEP_LINK=tressa://
SPEEDFLIGHT_AUTHOR=jakemor
SPEEDFLIGHT_ICON=App/Assets.xcassets/AppIcon.appiconset/icon-1024.png
EOF
chmod 600 .env.speedflight
grep -qxF '.env.speedflight' .gitignore || echo '.env.speedflight' >> .gitignore
```

`ASC_PRIVATE_KEY_PATH` is optional; the default is
`~/private_keys/AuthKey_$ASC_KEY_ID.p8`.

## 7. Write `scripts/speedflight.sh`

Write this script into the repo, filling in the UPPERCASE placeholders from
step 1. Commit it. It is the whole pipeline: archive, export, upload, link.

```bash
#!/bin/bash
# Cuts a signed ad hoc IPA, uploads it to Speedflight, and prints the page
# link as the last line. Signs through the App Store Connect key in
# .env.speedflight; the key must belong to DEVELOPMENT_TEAM.
#
#   scripts/speedflight.sh "<title>" "<notes>" [screenshot.png ...]
#
# The page link is the only auth for installing. The secret in
# .env.speedflight is the only auth for uploading. Do not paste either
# anywhere public.
set -euo pipefail
cd "$(dirname "$0")/.."

TITLE="${1:?usage: speedflight.sh \"<title>\" \"<notes>\" [screenshot.png ...]}"
NOTES="${2:?usage: speedflight.sh \"<title>\" \"<notes>\" [screenshot.png ...]}"
shift 2
SCREENSHOTS=("$@")

if [[ -f .env.speedflight ]]; then
  set -a
  # shellcheck disable=SC1091
  source .env.speedflight
  set +a
fi
: "${ASC_KEY_ID:?set ASC_KEY_ID in .env.speedflight}"
: "${ASC_ISSUER_ID:?set ASC_ISSUER_ID in .env.speedflight}"
: "${SPEEDFLIGHT_SECRET:?set SPEEDFLIGHT_SECRET in .env.speedflight}"
: "${SPEEDFLIGHT_DEEP_LINK:?set SPEEDFLIGHT_DEEP_LINK in .env.speedflight}"
: "${SPEEDFLIGHT_AUTHOR:?set SPEEDFLIGHT_AUTHOR in .env.speedflight}"
ASC_PRIVATE_KEY_PATH="${ASC_PRIVATE_KEY_PATH:-$HOME/private_keys/AuthKey_$ASC_KEY_ID.p8}"
[[ -f "$ASC_PRIVATE_KEY_PATH" ]] || { echo "missing ASC key file: $ASC_PRIVATE_KEY_PATH" >&2; exit 1; }

PROJECT="YOUR_APP.xcodeproj"      # or YOUR_APP.xcworkspace, with -workspace below
SCHEME="YOUR_SCHEME"
BUNDLE_ID="com.example.yourapp"
TEAM_ID="YOUR_TEAM_ID"
BASE="${SPEEDFLIGHT_BASE:-https://speedflight.dev}"
# The same worker on its workers.dev route, kept as an upload fallback. The
# page link stays on the custom domain.
FALLBACK_BASE="https://speedflight.jake-7c3.workers.dev"
OUT="build/share"

# The page shows a branch and commit, so those must be real: everything
# committed, and the commit on the remote. Under CI the checkout is the
# pushed commit by definition, and a detached HEAD has no upstream to test.
BRANCH="${GITHUB_REF_NAME:-$(git rev-parse --abbrev-ref HEAD)}"
COMMIT="$(git rev-parse HEAD)"
# https form of origin, so the page can link the branch and commit.
REPO_URL="$(git remote get-url origin 2>/dev/null | sed -E 's#^git@([^:]+):#https://\1/#; s#\.git$##')"
case "$REPO_URL" in https://*) ;; *) REPO_URL="" ;; esac
if [[ -z "${CI:-}" ]]; then
  if [[ -n "$(git status --porcelain)" ]]; then
    echo "working tree is dirty: commit before sharing a build" >&2
    exit 1
  fi
  if ! git merge-base --is-ancestor "$COMMIT" "@{u}" 2>/dev/null; then
    echo "HEAD is not pushed: git push -u origin $BRANCH" >&2
    exit 1
  fi
fi

# Uncomment for XcodeGen projects: the project file is generated and gitignored.
# xcodegen generate --quiet

rm -rf "$OUT"
mkdir -p "$OUT"

# Archive signed, not with CODE_SIGNING_ALLOWED=NO: an unsigned archive
# carries no entitlements and the export re-sign does not add them back.
# Cloud signing with the ASC key makes a development certificate for the
# archive and the ad hoc one for the export.
xcodebuild -project "$PROJECT" -scheme "$SCHEME" \
  -configuration Release \
  -destination "generic/platform=iOS" \
  -archivePath "$OUT/App.xcarchive" \
  -allowProvisioningUpdates \
  -authenticationKeyID "$ASC_KEY_ID" \
  -authenticationKeyIssuerID "$ASC_ISSUER_ID" \
  -authenticationKeyPath "$ASC_PRIVATE_KEY_PATH" \
  -quiet archive

cat > "$OUT/ExportOptions.plist" <<PLIST
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>destination</key><string>export</string>
  <key>method</key><string>release-testing</string>
  <key>signingStyle</key><string>automatic</string>
  <key>teamID</key><string>$TEAM_ID</string>
  <key>thinning</key><string>&lt;none&gt;</string>
</dict>
</plist>
PLIST

xcodebuild -exportArchive \
  -archivePath "$OUT/App.xcarchive" \
  -exportOptionsPlist "$OUT/ExportOptions.plist" \
  -exportPath "$OUT/export" \
  -allowProvisioningUpdates \
  -authenticationKeyID "$ASC_KEY_ID" \
  -authenticationKeyIssuerID "$ASC_ISSUER_ID" \
  -authenticationKeyPath "$ASC_PRIVATE_KEY_PATH" \
  -quiet
mv "$OUT"/export/*.ipa "$OUT/signed.ipa"

# 1. Register the build with its metadata. The server answers with the ids.
META="$(jq -n \
  --arg title "$TITLE" --arg notes "$NOTES" \
  --arg deepLink "$SPEEDFLIGHT_DEEP_LINK" \
  --arg branch "$BRANCH" --arg commit "$COMMIT" \
  --arg author "$SPEEDFLIGHT_AUTHOR" \
  --arg repoUrl "$REPO_URL" \
  '{title:$title, notes:$notes, deepLink:$deepLink, branch:$branch, commit:$commit, author:$author}
   + (if $repoUrl == "" then {} else {repoUrl:$repoUrl} end)')"
create() {
  curl -sfS --retry 3 --retry-all-errors --retry-delay 3 \
    -X POST "$1/api/apps/$SPEEDFLIGHT_SECRET/$BUNDLE_ID/builds" \
    -H "Content-Type: application/json" --data "$META"
}
CREATED="$(create "$BASE" || create "$FALLBACK_BASE")"
BUILD_ID="$(jq -r .buildId <<<"$CREATED")"
PAGE_URL="$(jq -r .pageUrl <<<"$CREATED")"

# 2. Upload the IPA. The server reads name, version, and build number from
#    its Info.plist and rejects it if the bundle id does not match.
upload() {
  curl -sfS --http1.1 --retry 5 --retry-all-errors --retry-delay 5 \
    -X PUT "$1/api/apps/$SPEEDFLIGHT_SECRET/$BUNDLE_ID/builds/$BUILD_ID/app.ipa" \
    --data-binary @"$OUT/signed.ipa" >/dev/null
}
upload "$BASE" || upload "$FALLBACK_BASE"

# 3. Screenshots, if given: what changed, as pictures. Named 01-, 02-, ...
#    so the page keeps the order you passed them in.
# The guarded expansion keeps macOS bash 3.2's set -u happy when no
# screenshots were passed; a bare "${SCREENSHOTS[@]}" aborts the script.
n=0
for shot in ${SCREENSHOTS[@]+"${SCREENSHOTS[@]}"}; do
  [[ -f "$shot" ]] || { echo "no such screenshot: $shot" >&2; continue; }
  n=$((n + 1))
  ext="${shot##*.}"
  case "$ext" in png|PNG) type=image/png ;; jpg|jpeg|JPG|JPEG) type=image/jpeg ;; webp) type=image/webp ;; *) echo "skip $shot: not png/jpg/webp" >&2; continue ;; esac
  name="$(printf '%02d-%s' "$n" "$(basename "$shot" | tr -c 'A-Za-z0-9._-\n' '-')")"
  curl -sfS --retry 3 --retry-all-errors --retry-delay 3 \
    -X PUT "$BASE/api/apps/$SPEEDFLIGHT_SECRET/$BUNDLE_ID/builds/$BUILD_ID/screenshots/$name" \
    -H "Content-Type: $type" --data-binary @"$shot" >/dev/null || echo "screenshot upload failed: $shot" >&2
done

# 4. Icon, if configured. Best effort.
if [[ -n "${SPEEDFLIGHT_ICON:-}" && -f "$SPEEDFLIGHT_ICON" ]]; then
  curl -sS -X PUT "$BASE/api/apps/$SPEEDFLIGHT_SECRET/$BUNDLE_ID/icon" \
    -H "Content-Type: image/png" --data-binary @"$SPEEDFLIGHT_ICON" >/dev/null || true
fi

echo
echo "Build page: $PAGE_URL"
```

Notes for you, the agent:

- `-destination "generic/platform=iOS"` always. Never address a phone.
- `release-testing` is the current name for ad hoc export. `thinning=<none>`
  keeps one universal IPA so it installs on any device in the profile.
- `-workspace` replaces `-project` in the archive call for workspaces.
- `jq` is required. `brew install jq` if missing.
- The script never installs on a device. Speedflight is for when the phone
  is not on the desk. Use the repo's own run-on-device script for that.

---

# Share a build (every time)

1. **Commit and push first.** The page shows the branch and commit, and the
   script refuses a dirty or unpushed tree. If the user's work is not
   committed, commit it with a real message and push. Ask before pushing to
   a branch that is not the user's own feature branch.
2. **Write the title and notes.** Look at the diff since the last shared
   build (`git log`, or the last Build page link in the conversation). The
   title is one line, what this build is. The notes say what changed and
   what to test, written for the person holding the phone. Plain text, a
   few short lines, no markdown headers.
3. **Screenshots (optional, encouraged).** The page shows them like the
   App Store, but they are not marketing. They show what changed: the
   screens you touched, before and after if that helps. Use what you have:
   screenshots the user gave you, or ones you took while testing on the
   simulator (`xcrun simctl io booted screenshot 01-paywall.png`). Do not
   start a simulator build just to get them. Any mix of iPhone and iPad,
   portrait and landscape, lays out fine; the page sets one height. PNG,
   JPG, or WebP, under 10MB each, at most 12.
4. **Run the script.**
   ```bash
   scripts/speedflight.sh "Onboarding paywall rewrite" "$(cat <<'EOF'
   What changed
   - New onboarding paywall with the annual plan first
   - Fixed the crash when restoring purchases offline

   What to test
   - Fresh install, go through onboarding, tap Restore with Wi-Fi off
   - Check the paywall shows 3 plans and the annual one is selected
   EOF
   )" shots/paywall.png shots/restore.png
   ```
   Screenshot paths come after the notes, in the order they should show.
   It takes 2 to 5 minutes. The last line is `Build page: <url>`.
5. **Post the link in the chat.** Every run, always, as a plain URL on its
   own line so it is tappable. The link is the app page, the `Build page:`
   line the script prints: `https://speedflight.dev/a/<pageId>`. Never
   share a URL with a build id after it (`/a/<pageId>/<buildId>`); that is
   one build's install page, and the user wants the page with all of them. Say what is on it: the title, version and
   build number, and that Install works from Safari on a registered iPhone.
   Do not paste the link anywhere public. Do not print the secret.

If the script fails:

- "No Accounts" or "No profiles": the ASC key does not belong to `TEAM_ID`,
  or lacks the App Manager role.
- "Could not read Info.plist" or "bundle id is X, but this page is for Y":
  the scheme built a different target than `BUNDLE_ID`. Fix the script's
  facts.
- "working tree is dirty" or "HEAD is not pushed": go back to step 1.
  Screenshots do not need to be committed; keep them out of git if they
  are throwaway.
- Upload timeouts: the script retries and falls back to the workers.dev host
  on its own. Run it again if both fail.

---

# On request: run it in GitHub Actions

Only when the user asks. The default is the Mac in front of them. The
reason to ask is a workflow with no Mac at all: an agent on a Linux devbox
or a cloud sandbox edits the iOS app, pushes, and GitHub's macOS runners
cut, sign, and upload the build. The user installs from the page link on
their phone. That is end-to-end iOS development from a machine that cannot
run Xcode.

The same `scripts/speedflight.sh` runs unchanged; it skips the dirty-tree
and pushed checks when `CI` is set.

## 1. Secrets and variables

```bash
gh secret set ASC_KEY_ID --body "ABC123DEFG"
gh secret set ASC_ISSUER_ID --body "12345678-abcd-...."
gh secret set ASC_PRIVATE_KEY < ~/private_keys/AuthKey_ABC123DEFG.p8
gh secret set SPEEDFLIGHT_SECRET --body "$(grep SPEEDFLIGHT_SECRET .env.speedflight | cut -d= -f2)"
gh variable set SPEEDFLIGHT_DEEP_LINK --body "tressa://"
```

Use the same `SPEEDFLIGHT_SECRET` as the local `.env.speedflight`, so local
and CI builds land on one page.

## 2. Workflow

Write `.github/workflows/speedflight.yml`:

```yaml
name: Speedflight

on:
  workflow_dispatch:
    inputs:
      title:
        description: One line, what this build is
        required: true
      notes:
        description: What changed and what to test
        required: true
  push:
    branches: ["**"]

concurrency:
  group: speedflight-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    runs-on: macos-latest
    timeout-minutes: 45
    env:
      ASC_KEY_ID: ${{ secrets.ASC_KEY_ID }}
      ASC_ISSUER_ID: ${{ secrets.ASC_ISSUER_ID }}
      SPEEDFLIGHT_SECRET: ${{ secrets.SPEEDFLIGHT_SECRET }}
      SPEEDFLIGHT_DEEP_LINK: ${{ vars.SPEEDFLIGHT_DEEP_LINK }}
      SPEEDFLIGHT_AUTHOR: ${{ github.actor }}
    steps:
      - uses: actions/checkout@v4
      - name: Install the ASC key
        run: |
          mkdir -p ~/private_keys
          printf '%s' "${{ secrets.ASC_PRIVATE_KEY }}" > ~/private_keys/AuthKey_${ASC_KEY_ID}.p8
      # Only for XcodeGen projects.
      - run: brew install xcodegen
      - name: Cut and share
        run: |
          TITLE="${{ inputs.title }}"
          NOTES="${{ inputs.notes }}"
          # On push, the commit message is the title and its body the notes.
          [ -n "$TITLE" ] || TITLE="$(git log -1 --format=%s)"
          [ -n "$NOTES" ] || NOTES="$(git log -1 --format=%b)"
          [ -n "$NOTES" ] || NOTES="$TITLE"
          scripts/speedflight.sh "$TITLE" "$NOTES" | tee build.log
          grep '^Build page:' build.log >> "$GITHUB_STEP_SUMMARY"
```

Notes for you, the agent:

- The page link grants installs. On a public repo, drop the step summary
  line; anyone can read it. The secret never prints.
- A fresh runner has no Apple Development certificate, so cloud signing
  mints one per run, and Apple caps those. After roughly ten runs the
  archive fails with "reached the maximum number of certificates". The
  fix is one fixed identity imported from a `.p12` secret before the
  archive step; ask the user for it when that error appears, and never
  create or revoke certificates yourself.
- Screenshots in CI need a simulator run in the workflow. Skip them unless
  the user asks; the page works without.
- Tell the user to keep pushing normally. Every push to any branch cuts a
  build; narrow `branches:` if that is too much.

---

# API reference

Writes need the secret and bundle id. Reads need only the page id.

```
POST   /api/apps/:secret/:bundleId/builds
       JSON: {title, notes, deepLink, branch, commit, author}  all required
             {repoUrl}  optional https repo URL; links branch + author on the page
       -> 201 {buildId, pageId, pageUrl}
PUT    /api/apps/:secret/:bundleId/builds/:buildId/app.ipa      raw IPA bytes
       -> {ok, appName, shortVersion, buildVersion, size, pageUrl}
PUT    /api/apps/:secret/:bundleId/builds/:buildId/screenshots/:name   raw image, under 10MB
       name like 01-home.png (png, jpg, webp); at most 12 per build
PUT    /api/apps/:secret/:bundleId/icon                         raw PNG, under 2MB
DELETE /api/apps/:secret/:bundleId/builds/:buildId

GET    /api/pages/:pageId                          app + builds JSON
GET    /api/pages/:pageId/builds/:buildId          one build
GET    /api/pages/:pageId/builds/:buildId/app.ipa  download
GET    /api/pages/:pageId/builds/:buildId/manifest.plist   OTA manifest

Page to share:     https://speedflight.dev/a/:pageId   (this one, always)
One build's page:  https://speedflight.dev/a/:pageId/:buildId   (what the QR opens; do not share)
```

Secret format: 32 to 128 chars of `[A-Za-z0-9_-]`. Mint with
`openssl rand -hex 24`. The page id is
`printf '%s\n%s' "$BUNDLE_ID" "$SPEEDFLIGHT_SECRET" | shasum -a 256 | cut -c1-32`,
if you ever need it without a server round trip.
