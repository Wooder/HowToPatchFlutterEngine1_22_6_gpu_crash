# Whats this?

This is a description of how to compile the Flutter Engine for Flutter Release 1.22.6 (Stable) with a bugfix that prevents executing code on the GPU if the app is in background mode.
This version of the engine potentially resolves the following issues (not yet confirmed):
- https://github.com/flutter/flutter/issues/76068
- https://github.com/flutter/flutter/issues/74226
- https://github.com/flutter/flutter/issues/69102

There is a merge request that fixes the problem on the master branch (https://github.com/flutter/engine/pull/24503).
I transferred this fix manually for the flutter engine for 1.22.6

The changes I made (2 commits on Mar 2, 2021) can be seen here:
https://github.com/Wooder/engine/commits/flutter_engine_for_stable_1_22_6_with_gpu_disable_sync_switch_via_appstate_fix

# Compiling the patched Flutter engine for Flutter stable_1_22_6 for iOS

## Prepare the source code

1. install `depot_tools` (<http://commondatastorage.googleapis.com/chrome-infra-docs/flat/depot_tools/docs/html/depot_tools_tutorial.html#_setting_up>) (In case of problems: the setup of the Flutter Engine Dev environment is explained here: <https://github.com/flutter/flutter/wiki/Setting-up-the-Engine-development-environment>)
2. In this directory create a file named `.gclient` with the following content (the branch used is based on the Flutter engine used in 1_22_6 (engine revision 2f0af37152) on which the bugfix from gaaclarke (from commit 90c8015280344f1faf0127599351e264d4d9a9d7) was manually transferred. It also contains a bug fix in the DEP file (siehe <https://github.com/flutter/flutter/issues/72108#issuecomment-744065093>)):

  ```
  solutions = [ {
  "managed": False,
  "name": "src/flutter",
  "url": "git@github.com:Wooder/engine.git@flutter_engine_for_stable_1_22_6_with_gpu_disable_sync_switch_via_appstate_fix",
  "custom_deps": {},
  "deps_file": "DEPS",
  "safesync_url": "",
  },
  ]
  ```

3. `cd` to the `engine`-Directory and execute`gclient sync` (this fetches alle required sources and will take about 1 hour)

## Compiling the Flutter Engine

1. Prepare the directories: `cd` to the `engine/src`-directory and execute 
   
   For debug builds
   
   * iOS debug for devices `./flutter/tools/gn --ios --unoptimized`
   * host debug `./flutter/tools/gn --unoptimized`
   * iOS debug for simulator ` ./flutter/tools/gn --ios --unoptimized --simulator`(iOS simulator)
   
   For release builds
   
   * iOS release build: `./flutter/tools/gn --ios --runtime-mode=release --unoptimized`
   * host release build: `./flutter/tools/gn --runtime-mode=release --unoptimized` 
   
   (for details on compiling the engine see: <https://github.com/flutter/flutter/wiki/Compiling-the-engine>)
3. Compiling with ninja: 
  
  For debug builds
  
  * debug build for iOS-devices: `ninja -C out/ios_debug_unopt`
  * host debug (always needed): `ninja -C out/host_debug_unopt`
  * debug build for iOS-simulator: `ninja -C out/ios_debug_sim_unopt`
  
  For release builds
  
  * host release build: `ninja -C out/host_release`
  * iOS release build: `ninja -C out/ios_release/`
4. The out-arguments must be adapted in Step 1 & 2 for optimized build

## Using the locally compiled, patched Flutter Engine with Flutter

`cd`in the directory of your flutter app and execute 

1. For a debug build on an iOS Device `flutter run --local-engine-src-path="/Users/yourusername/src/engine/src" --local-engine="ios_debug_unopt"`.
2. For a debug build on an iOS simulator `flutter run --local-engine-src-path="/Users/yourusername/src/engine/src" --local-engine="ios_debug_sim_unopt"`.
3. For a release build:`flutter run --local-engine-src-path="/Users/yourusername/src/engine/src" --local-engine="--local-engine="ios_release_unopt"`. 

Now the app starts with the patched engine.

# Problems with release build
When you execute the following command (to build a release of the app) 

`flutter build ios --local-engine-src-path="/Users/xxx/src/engine/src" --local-engine="ios_release_unopt"`

you will likely get this error message:

 ```
 /Users/xxx/source_code/flutter_user/ios/Flutter/Flutter.framework/Flutter, building for iOS-armv7 but attempting to link with file built for iOS-arm64
    Undefined symbols for architecture armv7:
      "_OBJC_CLASS_$_FlutterStandardMessageCodec", referenced from:
          objc-class-ref in FlutterWebView.o
      "_OBJC_CLASS_$_FlutterError", referenced from:
          objc-class-ref in FLTWKNavigationDelegate.o
          objc-class-ref in FlutterWebView.o
      "_FlutterMethodNotImplemented", referenced from:
          -[FLTCookieManager handleMethodCall:result:] in FLTCookieManager.o
          ___83-[FLTWKNavigationDelegate
 ```
 This is caused by Issue https://github.com/flutter/flutter/issues/51989  which was fixed with Pull Request https://github.com/flutter/flutter/pull/73072. Unfortunately, the fix is not yet included in the flutter tools for stable 1.22.6, so we need a workaround.

## The workaround

The workarround is to build the app only for arm64 (all 64 bit iPhones better than iPhone 5/ iPhone 5c).

1. Add the following post install hook to your Podfile (mybe merge it with an existing post install hook - only one hook is allowed):
 ```
post_install do |installer|
  installer.pods_project.targets.each do |target|
    flutter_additional_ios_build_settings(target)
  end

  installer.pods_project.targets.each do |target|
     target.build_configurations.each do |config|
       config.build_settings['ONLY_ACTIVE_ARCH'] = 'YES'
     end
  end
 end 
 ```
2. In Xcode open your build settings and set "Build Active Architecture Only" to YES for Debug/Profile/Release builds
3. In Xcode open your build settings and set architectures the following way:  Remove "$ARCHS_STANDARD" and add "arm64"
4. `cd` into `~/src/engine/src/out/ios_release_unopt/clang_x64` (the directory of your self built flutter engine) and
   `cp gen_snapshot gen_snapshot_arm64` - Xcode expects the `gen_snapshot_arm64` however after compiling the file is named `gen_snapshot`(but is arm64)
4. Change to the source directory of your flutter app in which you want to run `flutter build ios ..."`
5. `flutter clean``
6. `flutter build ios --local-engine-src-path="/Users/xxx/src/engine/src" --local-engine="ios_release_unopt"`
7. Now a error shows up but Generated.xcconfig was created and `pod install` was executed (Have a look at `Generated.xcconfig` inside Xcode)
8. Building the app in Xcode: Open the .xcworspace of your app with Xcode and select "Any iOS Device (arm64)" as target
9. In the Xcode main menu: Product -> Archive
10. Do NOT update architectures if Xcode asks to do so
11. The IPA for your app should now be created successfully
