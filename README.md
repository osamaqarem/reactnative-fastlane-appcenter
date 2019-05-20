# rn-fastlane-appcenter

🚀 Simple fastlane setup for signing, building and uploading react-native apps to [appcenter](https://appcenter.ms).

Works ideally with new react-native projects.

## Requirements

Install fastlane

- [fastlane](https://docs.fastlane.tools/getting-started/ios/setup/)

## Getting Started

At the root of your react-native project: Setup fastlane:

`mkdir fastlane && cd fastlane && touch Fastfile`

Install the appcenter plugin from the root of the project:

`cd .. && fastlane add_plugin appcenter`

## `iOS`

### Signing

For new projects, we need to create a bundle ID and provisioning profile.

To create a new bundle ID, run:

`fastlane produce create -i`

To setup provisioning profile, create a **private** GIT repo and then run:

`fastlane match init`

Use git and provide the private git repo URL.

A Matchfile will be created. Open it and change the profile type (e.g. appstore, adhoc) and add your Apple developer account under `username`.

Now, add the following to the Fastfile:

    platform :ios do    
      desc 'Fetch certificates and provisioning profiles'
      lane :certificates do
        match(app_identifier: 'com.app.bundle')
      end
    end

Change the bundle ID to yours.

In the future, pass `readonly: true` to the `match` action. This would ensure the command wouldn't create new provisioning profiles just in case.

**Important**: Use the same git repo for all apps under the same apple developer. Use different branches within the repo for different apps.  

https://docs.fastlane.tools/actions/match/#important-use-one-git-branch-per-team

### Building

If this is a new project: Open the project in Xcode. Correct the bundle ID. Select a development team. Exit.

Add the following lane to iOS:

    desc 'Fetch certificates. Build the iOS application.'
    lane :build do
      certificates
      increment_build_number(xcodeproj: './ios/YourAppName.xcodeproj')
      gym(scheme: 'YourAppName', project: './ios/YourAppName.xcodeproj', export_method: 'development')
    end

Change the app name to yours.

Generated `.ipa` will be at the project root `./appName.ipa`

### Uploading

Add the following to the iOS lane:

    desc 'Fetch certificates, build and upload to appcenter.'
    lane :beta do
      build
      appcenter_upload(
        api_token: ENV["TEST_APPCENTER_API_TOKEN"],
        owner_name: ENV["TEST_APPCENTER_OWNER_NAME"],
        app_name: ENV["APPCENTER_APP_NAME"],
        ipa: ENV["APPCENTER_DISTRIBUTE_IPA"]
      )
    end

Specify the .ipa path.

Run `fastlane ios beta`. You're now on Appcenter!

For environment variables, see [below.](#environment-variables)

## `Android`

### Signing

Follow the Facebook guide for generating a signing key (Only until "Adding signing config to your app's gradle config
" step).

https://facebook.github.io/react-native/docs/signed-apk-android.html

Navigate to JDK binary. Find the binary with:

`usr/libexec/java_home`

Generate keystore. Rename the key and alias.

`sudo keytool -genkey -v -keystore my-release-key.keystore -alias my-key-alias -keyalg RSA -keysize 2048 -validity 10000`

Move the key to your `android/app` directory.

Create gradle variables by adding the following to `android/gradle.properties`:

    MYAPP_RELEASE_STORE_FILE=my-release-key.keystore
    MYAPP_RELEASE_KEY_ALIAS=my-key-alias
    MYAPP_RELEASE_STORE_PASSWORD=*****
    MYAPP_RELEASE_KEY_PASSWORD=*****

You can change the names.

Edit `android/app/build.gradle` and add the signing config and build variant:

    ...
    android {
        ...
        defaultConfig { ... }
        signingConfigs {
            release {
                if (project.hasProperty('MYAPP_RELEASE_STORE_FILE')) {
                    storeFile file(MYAPP_RELEASE_STORE_FILE)
                    storePassword MYAPP_RELEASE_STORE_PASSWORD
                    keyAlias MYAPP_RELEASE_KEY_ALIAS
                    keyPassword MYAPP_RELEASE_KEY_PASSWORD
                }
            }
        }
        buildTypes {
            release {
                ...
                signingConfig signingConfigs.release
            }
        }
    }
    ...

Change the variable names according to the previous step. Done.

### Building

Add the following for Android:

    platform :android do
      desc 'Build the Android application.'
      lane :build do
        gradle(task: 'clean', project_dir: 'android/')
        gradle(task: 'assemble', build_type: 'release', project_dir: 'android/')
      end
    end

The build type has to be consistent with the build variant in the previous step.

Generated `.apk` will be at `android/app/build/outputs/apk/release/app-release.apk`

### Uploading

Add the following to the Android lane:

    desc 'Build and upload to appcenter.'
    lane :beta do
    build
    appcenter_upload(
        api_token: ENV["TEST_APPCENTER_API_TOKEN"],
        owner_name: ENV["TEST_APPCENTER_OWNER_NAME"],
        app_name: ENV["APPCENTER_APP_NAME"],
        apk: ENV["APPCENTER_DISTRIBUTE_APK"]
        )
    end

## `Optional` Add NPM Scripts

    "scripts": {
      ...,
      "ios:beta": "fastlane ios beta",
      "android:beta": "fastlane android beta"
    }


## Environment Variables

If you don't provide environment variables, you will be prompted to enter them when you run the commands. **You must specify the apk/ipa file paths with respect to the root project directory, though. Else the commands will fail with some variant of:**

`[!] Couldn't find build file at path ''`

Appcenter API Token:

https://appcenter.ms/settings/apitokens

Fastlane .env variables:

https://docs.fastlane.tools/advanced/other/#environment-variables
https://github.com/microsoft/fastlane-plugin-appcenter
