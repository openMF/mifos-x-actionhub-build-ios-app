# KMP Build iOS App
This GitHub Action automates the process of building and archiving an iOS application using Fastlane. It supports secure code signing via Fastlane Match and handles the complete build workflow and artifact storage.

## Features
- Automated iOS app building and archiving

- Temporary keychain creation for secure, non-interactive builds

- Secure code signing using Fastlane Match in readonly mode

- Compressed IPA artifact upload

- Optimized for CI/CD pipelines

## Prerequisites
Before using this action, make sure you have:

- An iOS project with a valid Xcode configuration (.xcodeproj or .xcworkspace, schemes, targets)

- Fastlane set up in the repository with a Fastfile under the fastlane/ directory

- A build_ios lane defined in the Fastfile

- Signing certificates and provisioning profiles stored and encrypted via Fastlane Match

- Your SSH private key (used to access the repo) is **base64-encoded** and saved as a GitHub secret

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
Name | Description
-- | --
match_ssh_private_key | Base64-encoded SSH private key for Fastlane Match Git access (required)
match_password | Password used by Match to decrypt provisioning assets (required)
match_type | Type of provisioning profile (adhoc, appstore, development) (required)
git_url |	URL to the certificates Git repo (required)
git_branch |	Branch to fetch provisioning assets from (required)
app_identifier | The iOS app bundle ID (e.g., com.example.app) (required)
provisioning_profile_name |	Name of the provisioning profile to use (required)

## Usage

```yaml
name: KMP iOS Build and Archive

on:
  workflow_dispatch:

jobs:
  build_ios_app:
    name: Build iOS App
    runs-on: macos-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Build iOS App
        uses: openMF/mifos-x-actionhub-build-ios-app@v1.0.1
        with:
           match_ssh_private_key: ${{ secrets.MATCH_SSH_PRIVATE_KEY }}
           match_password: ${{ secrets.MATCH_PASSWORD }}
           git_url: 'git@github.com:your-org/your-certificates-repo.git'
           git_branch: 'master'
           match_type: 'adhoc'
           app_identifier: 'com.example.myapp'
           provisioning_profile_name: 'match AdHoc com.example.myapp'
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