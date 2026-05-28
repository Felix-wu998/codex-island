# Sparkle auto-update

CodexIsland ships with [Sparkle 2](https://sparkle-project.org). On launch and
on a daily cadence the app fetches the appcast (an XML file attached to the
**latest** GitHub Release), compares versions, and prompts the user to
download + install if a newer build is listed. Updates are verified with an
EdDSA signature so a hijacked URL alone can't deliver malware.

The feed URL is:
```
https://github.com/Felix-wu998/codex-island/releases/latest/download/appcast.xml
```
GitHub's `releases/latest/download/<asset>` endpoint always 302-redirects to
the asset on the most recent non-prerelease release.

## One-time maintainer setup

1. Vendor Sparkle (idempotent):
   ```sh
   ./scripts/setup-sparkle.sh
   ```
2. Generate the EdDSA keypair. The current project key is stored in the
   `SPARKLE_ED_PRIVATE_KEY` GitHub Actions secret; do not commit the private
   key:
   ```sh
   node -e 'const fs=require("fs"),c=require("crypto");const {privateKey,publicKey}=c.generateKeyPairSync("ed25519");const priv=privateKey.export({format:"der",type:"pkcs8"});const pub=publicKey.export({format:"der",type:"spki"});const i=priv.lastIndexOf(Buffer.from([4,32]));fs.writeFileSync("sparkle_ed_priv",priv.subarray(i+2,i+34).toString("base64")+"\\n",{mode:0o600});console.log(pub.subarray(pub.length-32).toString("base64"));'
   ```
3. The **public** key is hardcoded in `build.sh` as `SU_PUBLIC_KEY`. Public
   keys are not secrets — they're meant to ship inside distributed apps so
   Sparkle can verify update signatures. If you generate a new keypair,
   replace the constant in `build.sh` and read the rotation warning below.
4. Store the **private** key for CI use:
   ```sh
   gh secret set SPARKLE_ED_PRIVATE_KEY -R Felix-wu998/codex-island < sparkle_ed_priv
   ```
   Then **delete `sparkle_ed_priv`** — never commit it.

The private key never leaves your Mac (and CI's runner). Lose it and existing
installs can no longer auto-update; you'd have to ship a new build with a
fresh public key embedded, which existing installs can't migrate to.

## Cutting a release

1. Bump `VERSION`.
2. Commit + tag + push:
   ```sh
   git commit -am "chore(release): bump VERSION to X.Y.Z" && git tag vX.Y.Z
   git push origin main vX.Y.Z
   ```
3. The release workflow runs `release.sh` on a macOS runner, which:
   - Builds a universal DMG
   - Signs it with the EdDSA key from the secret
   - Generates `dist/appcast.xml` listing the new version
   - Uploads **both** as release assets
   - Mirrors `Casks/codexisland.rb` to `Felix-wu998/homebrew-tap` with the tag
     version and freshly computed SHA-256, if `HOMEBREW_TAP_TOKEN` is configured

Existing installs pick up the update on their next daily check (or via
Settings → Updates → Check Now).

### Local dry-run

`./release.sh` works locally too. Without `$SPARKLE_PRIVATE_KEY_PATH`, it builds
the DMG and skips appcast signing. To test the full appcast path locally, point
`SPARKLE_PRIVATE_KEY_PATH` at a private-key file with the same format as the
GitHub secret. The DMG and appcast land in `dist/`, unpublished.

## Disabling the feature for a build

Set `SU_FEED_URL=` (empty) before running `build.sh`. Sparkle will still load
but won't have a feed to poll.
