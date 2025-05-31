# KMP Build iOS App
This GitHub Action automates the process of building and archiving an iOS application using Fastlane. It supports both **Debug and Release (signed)** builds, secure code signing via Fastlane Match, and handles complete artifact storage.

## Features
- Automated Debug and Release (signed) iOS app building and archiving

- Temporary keychain creation for secure, non-interactive builds

- Secure code signing using Fastlane Match for Release builds

- Compressed IPA artifact upload

- Optimized for CI/CD pipelines

## Prerequisites
Before using this action, make sure you have:

- An iOS project with a valid Xcode configuration (.xcodeproj or .xcworkspace, schemes, targets)

- Fastlane set up in the repository with a Fastfile under the fastlane/ directory

- A `build_ios` and `build_signed_ios` lanes defined in the Fastfile

- For Release builds: signing certificates and provisioning profiles stored and encrypted via Fastlane Match

- Your SSH private key (used to access the repo) is **base64-encoded** and saved as a GitHub secret

## Supported Build Types
| Build Type | Description                                                                  |
| ---------- | ---------------------------------------------------------------------------- |
| Debug      | Basic build for testing, no signing required                                 |
| Release    | Signed build with certificates via Fastlane Match, suitable for distribution |


## Step-by-Step: SSH Setup for Fastlane Match
1. Generate SSH Key Locally<br>
   In your terminal, run:
```yaml
ssh-keygen -t ed25519 -C "your_email@example.com"
```
It will ask for a file path. Press enter a custom path like:
```yaml
~/.ssh/match_ci_key
```
You can skip setting a passphrase when prompted (just hit enter twice).

This generates two files:

- `~/.ssh/match_ci_key` → Private key

- `~/.ssh/match_ci_key.pub` → Public key

2. Add the Private Key to the SSH Agent (optional but helpful)
   This step ensures the key is used during local development.
```yaml
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/match_ci_key
```
3. Add the Public Key to Your Certificates Repo (GitHub)
- Go to your certificates repo on GitHub (e.g., openMF/ios-provisioning-profile).
- Go to Settings → Deploy Keys.
- Click “Add deploy key”.
- Set title as `CI Match SSH Key` and paste the content of:

```yaml
cat ~/.ssh/match_ci_key.pub
```
- Check **Allow write access**.
- Click Add key.

4. Convert the Private Key to Base64
   This is how we pass it to GitHub Actions securely.
```yaml
base64 -i ~/.ssh/match_ci_key | pbcopy
```
This command copies the base64-encoded private key to your clipboard (macOS).

5. Save the Private Key as a GitHub Secret
- Go to the repo with your Fastfile (the project repo, not the certs repo).
- Navigate to Settings → Secrets and variables → Actions → New repository secret.
- Add the following:
    - Name: MATCH_SSH_PRIVATE_KEY
    - Value: Paste the base64-encoded private key

## Inputs
| Name            | Description                                      | Required    | Notes                |
| --------------- | ------------------------------------------------ | ----------- | -------------------- |
| build_type      | Build type: `Debug` or `Release`                 | No          | Default: `Debug`     |
| appstore_key_id | App Store Connect API key ID                     | Release only | Required for Release |
| appstore_issuer_id | App Store Connect issuer ID                      | Release only | Required for Release |
| appstore_auth_key | Base64-encoded `.p8` API private key             | Release only | Required for Release |
| match_ssh_private_key | Base64-encoded SSH private key for Match         | Release only | Required for Release |
| match_password  | Password to decrypt Match assets                 | Release only | Required for Release |
| match_type      | Profile type: `adhoc`, `appstore`, `development` | Release only | Required for Release |
| git_url         | URL of the Match cert repo                       | Release only | Required for Release |
| git_branch      | Branch in Match repo                             | Release only | Required for Release |
| app_identifier  | App bundle ID (`com.example.app`)                | Release only | Required for Release |
| provisioning_profile_name | Name of provisioning profile                     | Release only | Required for Release |


## Usage Example: Combined Debug & Release Workflow
```yaml
name: KMP iOS Build and Archive

  workflow_dispatch:
    inputs:
      build_type:
        type: choice
        options:
          - Release
          - Debug
        default: Debug
        description: Release Type

jobs:
  build_debug_ios_app:
    name: Build iOS App
    if: ${{ inputs.build_type == 'Debug' }}
    runs-on: macos-latest

    steps:
      - name: Set Xcode version
        uses: maxim-lobanov/setup-xcode@v1
        with:
          xcode_version: latest-stable

      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Build iOS App
        uses: openMF/mifos-x-actionhub-build-ios-app@v1.0.2
        with:
          build_type: ${{ inputs.build_type }}

  build_signed_ios_app:
    name: Build signed iOS app
    if: ${{ inputs.build_type == 'Release' }}
    runs-on: macos-latest

    steps:
      - name: Set Xcode version
        uses: maxim-lobanov/setup-xcode@v1
        with:
          xcode_version: latest-stable

      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Build signed iOS App
        uses: openMF/mifos-x-actionhub-build-ios-app@v1.0.2
        with:
          build_type: ${{ inputs.build_type }}
          git_url: 'git@github.com:your-org/your-certificates-repo.git'
          git_branch: 'master'
          match_type: 'adhoc'
          app_identifier: 'com.example.myapp'
          provisioning_profile_name: 'match AdHoc com.example.myapp'
          appstore_key_id: ${{ secrets.APPSTORE_KEY_ID }}
          appstore_issuer_id: ${{ secrets.APPSTORE_ISSUER_ID }}
          appstore_auth_key: ${{ secrets.APPSTORE_AUTH_KEY }}
          match_password: ${{ secrets.MATCH_PASSWORD }}
          match_ssh_private_key: ${{ secrets.MATCH_SSH_PRIVATE_KEY }}
```

## Fastlane Configuration
This Action requires the following Fastlane lanes to be defined in your Fastfile:

### Debug Build Lane
```yaml
desc "Build iOS application (Debug)"
lane :build_ios do |options|
    build_ios_app(
        scheme: "iosApp",
        project: "cmp-ios/iosApp.xcodeproj",
        output_name: "iosApp",
        output_directory: "cmp-ios/build",
        skip_codesigning: true,
        skip_archive: true
    )
end
```
### Release (Signed) Build Lane
```yaml
desc "Build Signed iOS application (Release)"
lane :build_signed_ios do |options|
    
    # CI Setup
    unless ENV['CI']
        UI.message("🖥️ Running locally, skipping CI-specific setup.")
    else
        setup_ci(provider: "circleci")
    end

    # Load API Key for App Store Connect
    app_store_connect_api_key(
        key_id: options[:appstore_key_id] || "DEFAULT_KEY_ID",
        issuer_id: options[:appstore_issuer_id] || "DEFAULT_ISSUER_ID",
        key_filepath: options[:key_filepath] || "secrets/Auth_key.p8",
        duration: 1200
    )

    # Fetch certificates and profiles with Match
    match(
        type: options[:match_type] || "adhoc",
        app_identifier: options[:app_identifier] || "org.mifos.kmp.template",
        readonly: false,
        git_url: options[:git_url] || "git@github.com:openMF/ios-provisioning-profile.git",
        git_branch: options[:git_branch] || "master",
        git_private_key: options[:git_private_key] || "~/.ssh/match_ci_key",
        force_for_new_devices: true,
        api_key: Actions.lane_context[SharedValues::APP_STORE_CONNECT_API_KEY]
    )

  app_identifier = options[:app_identifier] || "org.mifos.kmp.template"
  provisioning_profile_name = options[:provisioning_profile_name] || "match AdHoc org.mifos.kmp.template"
  # Build signed IPA
    build_ios_app(
        scheme: "iosApp",
        project: "cmp-ios/iosApp.xcodeproj",
        output_name: "iosApp",
        output_directory: "cmp-ios/build",
        export_options: {
          provisioningProfiles: {
            options[:app_identifier] => options[:provisioning_profile_name]
          }
        },
        xcargs: "CODE_SIGN_STYLE=Manual CODE_SIGN_IDENTITY=\"Apple Distribution\" DEVELOPMENT_TEAM=L432S2FZP5 PROVISIONING_PROFILE_SPECIFIER=\"#{options[:provisioning_profile_name]}\""
    )
end
```

## Workflow Details

1. **Environment Setup**
    - Configures Ruby environment
    - Sets up Fastlane with required plugins
    - Configures bundler for dependency management

2. **Build Process**
    - Executes Fastlane commands to build the iOS application
    - Generates IPA file
    - Handles build configuration through Fastlane

3. **Artifact Management**
    - Uploads built IPA file as an artifact
    - Applies high compression (level 9)
    - Sets 1-day retention period
    - Captures all IPA files in the build directory

## Requirements

- GitHub Actions runner with macOS
- Xcode installation
- Fastlane configuration in the repository

## Dependencies

- Ruby (installed via setup-ruby action)
- Bundler 2.2.27
- Fastlane with following plugins:
    - firebase_app_distribution
    - increment_build_number