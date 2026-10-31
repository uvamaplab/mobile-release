# mobile-release

A shared GitHub Actions workflow that builds an Expo app for iOS and Android, signs it, and uploads it for internal testing:

- **iOS** goes to App Store Connect, where the build appears in TestFlight.
- **Android** goes to Google Play's internal testing track.

It builds on the lab's self-hosted Mac when that is online, and on GitHub's runners when it is not.

[`.github/workflows/expo-release.yml`](.github/workflows/expo-release.yml) is the workflow. Its header explains what each step does and why.

## Using it in an app

Add `.github/workflows/release.yml` to the app:

```yaml
name: Phone Release

run-name: >-
  ${{ (github.ref_name == 'main' || startsWith(github.ref_name, 'release/') || startsWith(github.ref_name, 'hotfix/'))
  && format('Signed build from {0}', github.ref_name)
  || format('Unsigned build from {0}', github.ref_name) }}

on:
  workflow_dispatch:

permissions:
  contents: read
  id-token: write

jobs:
  release:
    uses: uvamaplab/mobile-release/.github/workflows/expo-release.yml@v1
    with:
      build-number-offset: 100
    secrets: inherit
```

To release, open **Actions > Phone Release > Run workflow** and pick a branch.

- **Signed runs.** A run on `main`, `release/*` or `hotfix/*` is signed and uploaded to both stores.
- **Unsigned runs.** A run on any other branch only checks that the app builds.

The build number is the run number plus `build-number-offset`, and both platforms share it.

- **Choosing the offset.** Set it above every build number the app has already uploaded, from EAS or anywhere else.
- **Don't rename the file.** GitHub counts runs per workflow file. If you rename `release.yml`, raise the offset at the same time.

### Inputs

| Input | Default | |
|---|---|---|
| `build-number-offset` | (required) | Added to the run number to make the build number |
| `xcode` | `26.6` | The Xcode at `/Applications/Xcode_<version>.app` |
| `java` | `17` | The JDK for Gradle |
| `node` | `22` | Node.js for both builds |

## Adding an app

### In the app's repository

1. **Create a `release` environment.** In Settings > Environments, add one whose deployment branches are `main`, `release/*` and `hotfix/*`.
2. **Add the app's own secrets to that environment:**
   - `IOS_PROVISIONING_PROFILE_BASE64`: the App Store provisioning profile for the app's bundle identifier, base64-encoded.
   - `ANDROID_KEYSTORE_BASE64`: the app's Play upload key, as a single-password PKCS12 keystore, base64-encoded.
   - `ANDROID_KEYSTORE_PASSWORD`: that keystore's password.

   Keep these in the environment. As repository or organization secrets, every branch would sign and upload.
3. **Set the Android package.** `app.json` needs `expo.android.package`, and `expo.ios.bundleIdentifier` must match the profile.

### In the uvamaplab organization

These are shared by every app. Each new app repository has to be added to them:

- **Organization secrets:** `IOS_CERTIFICATE_BASE64`, `IOS_CERTIFICATE_PASSWORD`, `APP_STORE_CONNECT_PRIVATE_KEY`, `APP_STORE_CONNECT_KEY_ID`, `APP_STORE_CONNECT_ISSUER_ID` and `RUNNER_STATUS_TOKEN`.
- **Organization variables:** `GOOGLE_PLAY_WORKLOAD_IDENTITY_PROVIDER`, `GOOGLE_PLAY_SERVICE_ACCOUNT`, `IOS_RUNNER` and `ANDROID_RUNNER`.
- **Runner group:** the group the Mac's runners belong to. It must allow the app's repository, and this workflow (`uvamaplab/mobile-release/.github/workflows/expo-release.yml@refs/tags/v1`). An app the group doesn't allow waits in the queue instead of falling back to GitHub's runners.

`RUNNER_STATUS_TOKEN` is a fine-grained token with the organization permission "Self-hosted runners: read". The workflow uses it to see whether the Mac is online.

### Apple

- **App ID:** register the app's bundle identifier on team L8C6JE7MP8, with the capabilities the app uses, such as HealthKit or Push Notifications.
- **Provisioning profile:** create an **App Store** profile for that App ID, with the team's Apple Distribution certificate.
- **App Store Connect:** create the app's record. Uploads need one to exist.

### Google

1. **Create the app** in Play Console with the package name from `app.json`.
2. **Upload the first bundle by hand**, using the `.aab` from a signed run's artifacts. Google only accepts API uploads once an app has a bundle. That first upload also registers the keystore's key as the app's upload key.
3. **Give the service account access.** In Users and permissions, give `GOOGLE_PLAY_SERVICE_ACCOUNT` access to the app, with "Release to testing tracks".
4. **Let the repository sign in to Google.** Add the repository's numeric ID to the Workload Identity provider's attribute condition, and to the service account's `roles/iam.workloadIdentityUser` binding, in the `digitaltrails-release` Google Cloud project. `gh api repos/uvamaplab/<repo> --jq .id` gives the ID.

## Changing the workflow

Apps use the `v1` tag.

- **To test a change**, call it from an app's branch at `@<branch>` or `@<sha>`.
- **To release a compatible change**, move `v1` to it. Every app then picks it up on its next run.
- **For a change that needs callers to change**, start a `v2` tag instead.

Also add the tested refs to the runner group's allowed workflows. Otherwise test runs can't use the Mac.
