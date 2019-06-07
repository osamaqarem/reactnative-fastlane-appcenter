# reactnative-fastlane-appcenter

🚀 Simple fastlane guide for signing, building and uploading new react-native apps to [Visual Studio App Center](https://appcenter.ms).


## Requirements

Install fastlane:

- [fastlane](https://docs.fastlane.tools/getting-started/ios/setup/)

## Getting Started

Create a react-native project:

`react-native init YourAppName`

At the root of your react-native project: Setup fastlane:

`cd YourAppName && mkdir fastlane && cd fastlane && touch Fastfile`

Install the App Center plugin from the root of the project:

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

Change `com.app.bundle` to the bundle ID you created earlier.

In the future, pass `readonly: true` to the `match` action. This would ensure the command wouldn't create new provisioning profiles.

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

Generated `.ipa` will be at the project root `./YourAppName.ipa`

### Uploading

Add the following to the iOS lane:

    desc 'Fetch certificates, build and upload to App Center.'
    lane :beta do
      build
      appcenter_upload(
        api_token: ENV["TEST_APPCENTER_API_TOKEN"],
        owner_name: ENV["TEST_APPCENTER_OWNER_NAME"],
        app_name: ENV["APPCENTER_APP_NAME"],
        ipa: ENV["APPCENTER_DISTRIBUTE_IPA"]
      )
    end

Specify the .ipa path. Following this guide it would be `ipa: "./YourAppName.ipa"`

Run `fastlane ios beta`. You're now on App Center! 👏

For environment variables, see [below.](#environment-variables)

## `Android`

### Signing

Navigate to JDK binary. Find the binary with:

`/usr/libexec/java_home`

Generate keystore. Rename the key and alias.

`sudo keytool -genkey -v -keystore my-release-key.keystore -alias my-key-alias -keyalg RSA -keysize 2048 -validity 10000`

Move the key to your `android/app` directory.

`sudo mv my-release-key.keystore [...path to YourAppName/android/app]`

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

    desc 'Build and upload to App Center.'
    lane :beta do
    build
    appcenter_upload(
        api_token: ENV["TEST_APPCENTER_API_TOKEN"],
        owner_name: ENV["TEST_APPCENTER_OWNER_NAME"],
        app_name: ENV["APPCENTER_APP_NAME"],
        apk: ENV["APPCENTER_DISTRIBUTE_APK"]
        )
    end

Specify the .apk path. Following this guide it would be `apk: "./android/app/build/outputs/apk/release/app-release.apk"`

Run `fastlane android beta`. You're now on App Center! 👏

## `Optional` Add NPM Scripts

    "scripts": {
      ...,
      "ios:beta": "fastlane ios beta",
      "android:beta": "fastlane android beta"
    }


## Environment Variables

If you don't provide environment variables, you will be prompted to enter them when you run the commands. You must specify the apk/ipa file paths with respect to the root project directory **and not with respect to the fastfile location.** Else the commands will fail with some variant of:

`[!] Couldn't find build file at path ''`

Fastlane .env variables:

https://docs.fastlane.tools/advanced/other/#environment-variables

## Useful Links

App Center API Token:

https://appcenter.ms/settings/apitokens

Generating Signed Android APK:

https://facebook.github.io/react-native/docs/signed-apk-android.html

App Center fastlane Plugin

https://github.com/microsoft/fastlane-plugin-appcenter

Other fastlane Guides:

https://docs.fastlane.tools/getting-started/cross-platform/react-native/

## Todo

- Add instructions for building on App Center from a repo.
- A more automated way of signing for android.
