# iOS code signing on ephemeral runners: a field guide to not leaking your certificates

<!-- canonical: https://github.com/macrunara/docs/blob/main/blog/03-ephemeral-runner-ios-signing/index.en.md -->
<!-- Status: DRAFT. Verify every command against the example repos before publishing. -->

*Code signing concentrates your most dangerous credentials onto machines that execute untrusted code. On ephemeral infrastructure this is a feature, not a bug — if you set it up right. Here's the threat model and the exact setup we recommend.*

---

Think about what an iOS release pipeline touches: your **distribution certificate's private key** (impersonate your company to every iPhone on Earth), your **App Store Connect credentials** (submit apps, change metadata, in worst cases withdraw builds), and your **notarization credentials** for macOS. Now think about where it runs: a CI machine that, five minutes earlier, executed `pod install` scripts from a dependency you've never audited.

The industry answer has converged on three ideas. Ephemeral runners make all three stronger:

1. **Prefer API keys over passwords.** App Store Connect API keys are scoped, revocable, and don't require 2FA bypasses.
2. **Contain secrets in time and space.** A secret should exist on disk for the shortest possible window, inside the smallest possible boundary.
3. **Make the machine disposable.** The strongest cleanup is `rm -rf` on the whole computer.

## Threat model, briefly

| Threat | Mitigated by |
|---|---|
| Malicious dependency exfiltrates the signing key mid-build | Secrets only enter the keychain in the signing step, not for the whole job; private key never written unencrypted |
| Later job on the same machine scavenges leftovers | One job per VM; the VM — keychain included — is destroyed |
| Stolen long-lived credential reused from attacker's machine | App Store Connect API key (revocable in one click, no password reset cascade) |
| Over-privileged token | ASC keys with the minimum role; GitHub environments with required reviewers on release jobs |
| CI logs leaking secrets | Key material passed via stdin/env, never as command arguments; `set +x` hygiene |

## The golden rule: App Store Connect API keys for everything

Everything in a modern Apple release pipeline — TestFlight upload, notarization, certificate/profile provisioning via fastlane — can authenticate with an **App Store Connect API key**. One key type replaces: Apple ID + password + app-specific password + 2FA SMS gymnastics.

```
App Store Connect → Users and Access → Integrations → App Store Connect API
→ Team Keys → (+) → role: App Manager (sufficient for signing + TestFlight)
```

Store three repository secrets: `KEY_ID`, `ISSUER_ID`, and the `.p8` content (base64). The key ID and issuer ID are not high-entropy secrets; the `.p8` is — treat it accordingly, and note that **Apple only lets you download the `.p8` once**. Rotate on a schedule (quarterly is cheap insurance) and instantly on suspicion.

## The ephemeral keychain dance (macOS / Developer ID flow)

When you must import a `.p12` — the Developer ID flow for macOS distribution — do it inside a throwaway keychain, in a throwaway VM:

```bash
set -euo pipefail
# Secrets arrive via environment, never argv:
#   MACOS_CERTIFICATE_P12_BASE64, MACOS_CERTIFICATE_PASSWORD, KEYCHAIN_PASSWORD
echo "$MACOS_CERTIFICATE_P12_BASE64" | base64 --decode > certificate.p12

security create-keychain -p "$KEYCHAIN_PASSWORD" ci.keychain-db
security default-keychain -s ci.keychain-db
security unlock-keychain -p "$KEYCHAIN_PASSWORD" ci.keychain-db

# -T grants codesign non-interactive access to this keychain only
security import certificate.p12 -k ci.keychain-db \
  -P "$MACOS_CERTIFICATE_PASSWORD" -T /usr/bin/codesign

# Without this line, codesign prompts for the keychain password mid-build
# and your headless job hangs forever
security set-key-partition-list -S apple-tool:,apple:,codesign: \
  -s -k "$KEYCHAIN_PASSWORD" ci.keychain-db

shred -u certificate.p12 2>/dev/null || rm -f certificate.p12
```

Then sign with an explicit identity and a **secure timestamp** (`notarytool` rejects timestamp-less signatures):

```bash
xcodebuild -scheme NotarizeExample -configuration Release \
  CODE_SIGN_STYLE=Manual CODE_SIGN_IDENTITY="Developer ID Application" \
  DEVELOPMENT_TEAM="$MACOS_TEAM_ID" OTHER_CODE_SIGN_FLAGS="--timestamp" \
  ENABLE_HARDENED_RUNTIME=YES build
```

Notarize with the same API-key pattern — no Apple ID anywhere:

```bash
xcrun notarytool submit app.dmg \
  --key AuthKey.p8 --key-id "$KEY_ID" --issuer "$ISSUER_ID" \
  --wait --timeout 30m
xcrun stapler staple app.dmg
```

The complete, runnable version of this pipeline is in our [macos-notarize-example](https://github.com/macrunara/macos-notarize-example) repo — fork it, set seven secrets, push a tag.

## iOS / TestFlight: cert + sigh with the API key, no match repo required

fastlane's `match` is the classic answer for team-wide signing, but it adds a certificates git repo to your threat surface. For CI-only distribution, `cert` + `sigh` driven by the API key is leaner:

```ruby
api_key = app_store_connect_api_key(
  key_id: ENV["APP_STORE_CONNECT_API_KEY_ID"],
  issuer_id: ENV["APP_STORE_CONNECT_API_ISSUER_ID"],
  key_content: ENV["APP_STORE_CONNECT_API_KEY_CONTENT"],
  is_key_content_base64: true
)
cert(api_key: api_key)
sigh(api_key: api_key, app_identifier: "com.example.app", force: true)
build_app(scheme: "App", export_method: "app-store")
upload_to_testflight(api_key: api_key)
```

Full lane in [ios-swift-example](https://github.com/macrunara/ios-swift-example). Two notes:

- `cert` needs to create/login to the Developer portal with the same API key; the **App Manager** role covers it. If your organization separates portal roles tightly, use `match` with a read-only CI deploy key instead — also fine.
- The Release configuration of your project should be `CODE_SIGN_STYLE = Manual`; let Debug stay Automatic so local development never touches this machinery.

## Why ephemerality is the security feature

Every mitigation above has a failure mode where something is left behind: a keychain file, an unlocked session, a `.p8` someone echoed into a build log that's still on disk. On persistent self-hosted runners, those accumulate forever, across every customer's jobs.

On Macrunara the cleanup is structural: when your job ends, **the VM it ran in is destroyed** — keychain, login session, DerivedData, shell history, everything. The next job clones a fresh VM from the golden image. "Did we remember to delete the secret?" stops being a question that matters, because there is nothing left to delete from.

Combined with GitHub-side hygiene — environments with required reviewers on the release workflow, secrets scoped to those environments, `ACTIONS_STEP_DEBUG` off in production — you get a pipeline where the worst-case credential exposure window is *the duration of a single job, inside a single VM, with a revocable key*.

## Checklist

- [ ] ASC API key for TestFlight + notarization + cert/sigh; Apple ID password nowhere in CI
- [ ] Secrets in repository/environment secrets, release jobs behind required reviewers
- [ ] Throwaway keychain per job (`create-keychain` → `set-key-partition-list` → sign)
- [ ] Key material via env/stdin, never argv; `.p12` deleted in the same step that imports it
- [ ] `OTHER_CODE_SIGN_FLAGS="--timestamp"` for anything headed to notarytool
- [ ] Ephemeral runner per job — no warm, stateful machines in the signing path
- [ ] Key rotation calendar entry (quarterly)

---

*This is part 3 of our series on macOS CI internals. Part 1 covered the [cluster architecture](https://github.com/macrunara/docs/blob/main/blog/01-m4-cluster-architecture/index.en.md); part 2, the [build-cache consistency model](https://github.com/macrunara/docs/blob/main/blog/02-xcode-build-cache-consistency/index.en.md). Next up: the TCO spreadsheet. Tools and further reading: [awesome-macos-ci](https://github.com/macrunara/awesome-macos-ci).*
