# Repro: Expo LogBox shows a black screen on Android `debugOptimized`

Minimal Expo SDK 58 app for a bug in `@expo/log-box` (58.0.15).

With `EXPO_UNSTABLE_LOG_BOX=1`, the app JavaScript uses the Expo LogBox, which loads
`file:///android_asset/ExpoLogBox.bundle/index.html` in a WebView on Android.
`packages/@expo/log-box/android/build.gradle` adds that bundle to the `debug` source set only.
React Native adds a `debugOptimized` build type to the app. `@expo/log-box` has no such build type,
so the app's `debugOptimized` variant uses the library's `release` variant, which has no bundle.

## Steps

Prerequisites: Android SDK, JDK 17, an Android emulator or device, `adb`.

```sh
npm install
export EXPO_UNSTABLE_LOG_BOX=1          # PowerShell: $env:EXPO_UNSTABLE_LOG_BOX = "1"
npx expo prebuild --platform android --no-install
npx expo start                          # keep running; the variable is also in the `.env` file
```

In a second terminal (the variable must also be set for Gradle):

```sh
cd android
./gradlew :app:assembleDebugOptimized -PreactNativeArchitectures=x86_64
adb install -r app/build/outputs/apk/debugOptimized/app-debugOptimized.apk
adb reverse tcp:8081 tcp:8081
adb shell am start -n com.example.logboxrepro/.MainActivity
```

Then:

1. Check the APK: `unzip -l app/build/outputs/apk/debugOptimized/app-debugOptimized.apk | grep ExpoLogBox`.
2. In the app, press "Trigger a LogBox error". A LogBox toast appears.
3. Tap the toast.

`npx expo run:android --variant debugOptimized` builds the same variant. The reporter did not run it
for this repro.

## Expected

The Expo LogBox inspector opens, with the error, the code frame and the call stack.
The APK contains `assets/ExpoLogBox.bundle/`.

## Actual

The inspector is a black screen with an Android robot icon (the WebView error page).
`adb logcat` prints `Unable to open asset URL: file:///android_asset/ExpoLogBox.bundle/index.html`.
The APK has no `ExpoLogBox.bundle` files.
The `debug` variant was not built for this repro. It is the variant that `@expo/log-box` adds
the bundle to.

## Notes

- `@expo/log-box` is listed in `package.json` on purpose. Without it, npm installs the package under
  `node_modules/expo/node_modules/`, and Expo CLI cannot resolve `@expo/log-box/swap-rn-logbox.js`
  from the app root. Then the Expo LogBox is not enabled in the JavaScript bundle and React Native's
  own LogBox shows instead.
- The steps above use the default Metro port 8081. The reporter ran Metro on port 8082 with
  `-PreactNativeDevServerPort=8082` and a `debug_http_host` preference set to `localhost:8082`.
- Only Android was tested. The iOS LogBox is not affected by this build script.
