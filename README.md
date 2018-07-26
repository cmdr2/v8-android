# v8-android
Example Android Studio project that embeds v8 (plus some notes on compiling v8 for android)

The intention is to embed v8 inside an Android app, and this project is an example of getting "Hello World" from Javascript to show up on the Android app (using v8).

**Note:** If you actually want to use Javascript inside your Android app, I recommend you use [J2V8](https://github.com/eclipsesource/J2V8) instead (unless you need to make 1000s of calls/sec between JS and the runtime, in which case you should use this project's approach to avoid the [JNI overhead in J2V8](https://github.com/eclipsesource/J2V8/issues/194#issuecomment-252958734)). This project is a learning exercise, or a starting point to start a v8 binding from scratch.

## Example uses for v8 in Android
I needed it for building an Android game, which used Javascript as a scripting language for modding. Other engines weren't performant-enough or embeddable. Your uses may vary.

*Disclaimer: This was a learning exercise, and the steps below were the ones that finally worked. I'm sure there is room to reduce the binary size and improve things. I followed [ibon's excellent guide](https://medium.com/@hyperandroid/compile-v8-for-arm-7-df45372f9d4e), but needed to change a few things to make it work.*

## How to use:
*Install the [jni-test.apk](https://github.com/cmdr2/v8-android/blob/master/jni-test.apk) file (~7mb) incase you only want to see it running. This apk has only been compiled for armeabi-v7a, so let me know if it doesn't work on your phone*

### In Android Studio:
1. Clone this repository, and open the `jni-test` directory as a project in your Android Studio.
2. Run the project after connecting your phone via USB.

Incase it has issues, try renaming the `app/libs/armeabi-v7a` directory to your phone's ABI format, like `app/libs/arm64-v8a` or something. Or maybe just edit the `app/CMakeLists.txt` file and remove the `${ANDROID_ABI}` parameter.

This uses a prebuilt v8 library for Android, compiled from the 6.8 branch of v8 on July 22, 2018 (5b8126a8d6fa0c58c89c2d618264ee087d6795a1 (HEAD -> 6.8, tag: lkgr/6.8, tag: 6.8.275.2)). You may want to create a new binary.

## Files of interest:
1. The compiled APK file, if you want to try: [jni-test.apk](https://github.com/cmdr2/v8-android/blob/master/jni-test.apk)  (~7mb)
2. The [CMakeLists.txt](https://github.com/cmdr2/v8-android/blob/master/jni-test/app/CMakeLists.txt) for linking the v8 libraries
3. The [native-lib.cc](https://github.com/cmdr2/v8-android/blob/master/jni-test/app/src/main/cpp/native-lib.cpp) C++ file that contains a 'Hello World' v8 program, and gets called by [MyActivity.java](https://github.com/cmdr2/v8-android/blob/master/jni-test/app/src/main/java/org/cmdr2/jnitest/MyActivity.java)
4. The v8 library binaries compiled for Android on July 20, 2018: [libv8_base.a](https://github.com/cmdr2/v8-android/blob/master/jni-test/app/libs/armeabi-v7a/libv8_base.a) and [libv8_snapshot.a](https://github.com/cmdr2/v8-android/blob/master/jni-test/app/libs/armeabi-v7a/libv8_snapshot.a).

## Here's what was done broadly:
1. Compile v8 for android_arm target
2. Manually generate the ".a" static libraries from the ".o" files using the 'ar' command. The build-generated versions weren't working for some reason.
3. Create an Android NDK project from Android Studio.
4. Copy the `include` directory from v8, and the static libraries created previously and paste into an Android project's `app/libs/armeabi-v7a/` directory.
5. Specify these libraries and include directory path in the Android project's `CMakesLists.txt` file.
6. Use v8 in the native C++ file, which gets called from the Java Android Activity via JNI.

## For compiling v8:
0. Build on Ubuntu (or some Linux with ELF and glibc). I was on a Mac, so I created a Virtual Machine and used Ubuntu Server for a minimal distro. Give it atleast 40 GB of disk (can use dynamic resize).
1. Follow https://github.com/v8/v8/wiki/Building-from-Source
2. But before running `v8gen.py`, first run `echo "target_os = ['android']" >> /path/to/workingDir/.gclient && gclient sync`
3. Then run `tools/dev/v8gen.py gen -m client.v8.ports -b "V8 Android Arm - builder" android_arm.release`
4. Next, run `gn args out.gn/android_arm.release` and use these flags:
```
is_component_build = false
is_debug = false
symbol_level = 1
target_cpu = "arm"
target_os = "android"
use_goma = false
v8_android_log_stdout = true
v8_static_library = true
use_custom_libcxx = false
use_custom_libcxx_for_host = false
v8_use_external_startup_data = false
```
5. Next, run `ninja -C out.gn/android_arm.release`
6. Then create the `libv8_base.a` and `libv8_snapshot.a` static libraries manually (the built ones don't seem to work for some reason):
```
mkdir libs
cd libs
ar -rcsD libv8_base.a /path/to/v8/out.gn/android_arm.release/obj/v8_base/*.o
ar -rcsD libv8_base.a /path/to/v8/out.gn/android_arm.release/obj/v8_libbase/*.o
ar -rcsD libv8_base.a /path/to/v8/out.gn/android_arm.release/obj/v8_libsampler/*.o
ar -rcsD libv8_base.a /path/to/v8/out.gn/android_arm.release/obj/v8_libplatform/*.o
ar -rcsD libv8_base.a /path/to/v8/out.gn/android_arm.release/obj/src/inspector/inspector/*.o
ar -rcsD libv8_base.a /path/to/v8/out.gn/android_arm.release/obj/third_party/icu/icuuc/*.o
ar -rcsD libv8_base.a /path/to/v8/out.gn/android_arm.release/obj/third_party/icu/icui18n/*.o
ar -rcsD libv8_snapshot.a /path/to/v8/out.gn/android_arm.release/obj/v8_snapshot/*.o
```
6. Then copy the includes using `cp -R /path/to/v8/include .`
7. Now you can use these in your Android NDK project

## Using the compiled v8 binaries in an Android NDK project
1. Either create a new Android NDK project in your Android Studio, or use one that's already set up with CMake
2. Copy the `include` directory from v8, and the static libraries created previously and paste into an Android project's `app/libs/armeabi-v7a/` directory.
3. Specify these libraries and include directory path in the Android project's `CMakesLists.txt` file.
4. Use v8 in the native C++ file, which gets called from the Java Android Activity via JNI.

Use the example project for the example `CMakeLists.txt` and C++ file.

## Random note
The key gotcha was that the `.a` files created by the build didn't seem to work for some reason, or maybe I was messing the order. So what worked was manually combining the relevant `.o` files (base, libbase, platform, sampler, inspector, icui18n, icuuc) into libv8_base.a, and the v8_snapshot `.o` files into libv8_snapshot.a.

Then the order in CMakeList was important: v8_base followed by v8_snapshot.

Another thing to remember: the v8 compilation process will probably take 30 GB of disk space, and download probably 15 GB of stuff. This could be because it downloaded stuff a few times, mostly because of my sheer incompetence at compiling v8. But size your VM disk, time and patience accordingly.
