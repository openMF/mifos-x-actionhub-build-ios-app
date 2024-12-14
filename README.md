# KMP Build iOS App

This GitHub Action automates the process of building and archiving an iOS application using Fastlane. It handles the complete build workflow and artifact storage.

## Features

- Automated iOS app building
- Fastlane integration
- Build artifact compression and storage
- Optimized for CI/CD pipelines

## Prerequisites

- iOS project with Xcode configuration
- Fastlane setup in the repository
- Valid iOS certificates and provisioning profiles
- Proper Xcode project structure

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
        uses: openMF/kmp-build-ios-app-action@v1.0.0
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
- Valid iOS development certificates
- Fastlane configuration in the repository

## Dependencies

- Ruby (installed via setup-ruby action)
- Bundler 2.2.27
- Fastlane with following plugins:
    - firebase_app_distribution
    - increment_build_number

