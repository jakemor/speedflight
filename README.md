# Speedflight

Get a local iOS build onto your phone without being at the Mac.

Your coding agent cuts a cloud-signed build, uploads it to
[speedflight.dev](https://speedflight.dev), and posts a link. Open the link
in Safari on any iPhone registered to your Apple account and tap Install.
Every build on the page has a title, release notes, the branch and commit,
and who cut it.

No accounts. The link is the only key, and Apple signs the app, so only
devices on the provisioning profile can install it.

## Install

Tell your agent:

```
setup speedflight.dev
```

Or by hand:

```bash
npx skills add jakemor/speedflight
```

Then, in your iOS repo, say `/speedflight`. The agent sets the repo up once
(App Store Connect key, upload secret, deep link, a small script), then cuts
and shares a build every time you ask, or whenever your phone is not
plugged in.

The skill is [skills/speedflight/SKILL.md](skills/speedflight/SKILL.md).
