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

1. Prepare the directories: `cd` to the `engine/src`-directory and execute `./flutter/tools/gn --ios --unoptimized` (iOS) and `./flutter/tools/gn --unoptimized`(host debug) and ` ./flutter/tools/gn --ios --unoptimized --simulator`(iOS simulator) (for details see: <https://github.com/flutter/flutter/wiki/Compiling-the-engine>)
2. Compiling with ninja: `ninja -C out/ios_debug_unopt && ninja -C out/host_debug_unopt && ninja -C out/ios_debug_sim_unopt`
3. The out-arguments must be adapted in Step 1 & 2 for optimized build

## Using the locally compiled, patched Flutter Engine with Flutter

`cd`in the directory of your flutter app and execute 

1. For a debug build on an iOS Device `flutter run --local-engine-src-path="/Users/yourusername/src/engine/src" --local-engine="ios_debug_unopt"`.
2. For a debug build on an iOS simulator `flutter run --local-engine-src-path="/Users/yourusername/src/engine/src" --local-engine="ios_debug_sim_unopt"`.
3. For a release build: TODO

Now the app starts with the patched engine.
