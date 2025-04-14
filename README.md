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

## Inputs
Name | Description
-- | --
match_git_basic_authorization | Base64-encoded GitHub token for accessing the signing repo (required)
match_password | Password used by Match to decrypt provisioning assets (required)
match_type | Type of provisioning profile (adhoc, appstore, development) (required)
keychain_name |	Name of the temporary keychain created during build (required)
keychain_password |	Password for unlocking the temporary keychain (required)
export_method | How the app is packaged (app-store, ad-hoc, etc.) (required)
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
          match_git_basic_authorization: ${{ secrets.MATCH_GIT_AUTH }}
          match_password: ${{ secrets.MATCH_PASSWORD }}
          match_type: "appstore"
          keychain_name: "ios-build-keychain"
          keychain_password: ${{ secrets.KEYCHAIN_PASSWORD }}
          export_method: "app-store"
          app_identifier: "com.example.myapp"
          provisioning_profile_name: "MyApp_AppStore_Profile"
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

