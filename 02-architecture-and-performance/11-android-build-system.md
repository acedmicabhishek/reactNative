# Android Build System for React Native

## Overview
React Native Android apps are built using **Gradle** — the standard Android build tool. Understanding the build pipeline from source code to `.apk`/`.aab` lets you optimize build times, control what ends up in your binary, and integrate C++ native code cleanly.

---

## 1. The Build Pipeline

```
Kotlin/Java Sources + C++ Sources + JS Bundle + Assets + AndroidManifest
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           GRADLE BUILD                                       │
│                                                                             │
│  1. Kotlin/Java compilation (kotlinc/javac → .class files)                 │
│  2. C++ compilation (CMake/ndk-build → .so shared libraries)               │
│  3. DEX compilation (R8/D8 → .dex files — ART bytecode)                    │
│  4. Resource compilation (aapt2 → compiled resources)                       │
│  5. Metro bundle generation (JS → index.android.bundle)                     │
│  6. APK/AAB packaging (zip + sign)                                          │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
app-release.aab (Android App Bundle) or app-release.apk
```

---

## 2. Project Structure

```
android/
├── app/
│   ├── src/
│   │   └── main/
│   │       ├── java/com/yourapp/       ← Kotlin/Java source
│   │       ├── cpp/                    ← C++ source files (if any)
│   │       ├── res/                    ← Android resources (layouts, drawables, strings)
│   │       └── AndroidManifest.xml     ← App configuration
│   ├── build.gradle                    ← App-level Gradle config
│   └── CMakeLists.txt                  ← C++ build config (if using native code)
├── gradle/
│   └── wrapper/
│       └── gradle-wrapper.properties   ← Gradle version
├── build.gradle                        ← Project-level Gradle config
├── gradle.properties                   ← Build properties & flags
└── settings.gradle                     ← Included modules
```

---

## 3. `build.gradle` (App Level)

```groovy
// android/app/build.gradle
android {
    compileSdk 34
    
    defaultConfig {
        applicationId "com.yourcompany.yourapp"
        minSdk 24                 // minimum Android version supported
        targetSdk 34              // API level app is designed for
        versionCode 1             // integer, increments on each release
        versionName "1.0.0"       // human-readable version string
        
        // Supported CPU architectures (affects .so file inclusion)
        ndk {
            abiFilters "armeabi-v7a", "arm64-v8a", "x86", "x86_64"
            // Production: drop x86/x86_64 (saves ~10MB) unless supporting emulators
            // abiFilters "armeabi-v7a", "arm64-v8a"
        }
    }
    
    buildTypes {
        debug {
            // Dev server connection
            buildConfigField "boolean", "IS_NEW_ARCHITECTURE_ENABLED", "false"
            signingConfig signingConfigs.debug
        }
        release {
            minifyEnabled true        // enable R8 minification and tree-shaking
            shrinkResources true      // remove unused resources
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
            signingConfig signingConfigs.release
        }
    }
    
    // C++ build configuration
    externalNativeBuild {
        cmake {
            path "CMakeLists.txt"
            version "3.22.1"
        }
    }
}
```

---

## 4. Gradle Properties — React Native Flags

```properties
# android/gradle.properties

# New Architecture
newArchEnabled=true

# Hermes JS engine
hermesEnabled=true

# Enable ProGuard-based code shrinking
android.enableR8.fullMode=true

# Build optimizations
org.gradle.jvmargs=-Xmx4g -XX:MaxMetaspaceSize=512m -XX:+HeapDumpOnOutOfMemoryError -Dfile.encoding=UTF-8
org.gradle.parallel=true
org.gradle.configureondemand=true
org.gradle.caching=true
android.useAndroidX=true
```

---

## 5. R8 / ProGuard — Code Shrinking & Optimization

R8 is the modern successor to ProGuard. It does four things:

1. **Shrinking (Tree-shaking):** Removes unused classes, methods, and fields.
2. **Minification:** Renames classes/methods/fields to short names (`a`, `b`, `c`...).
3. **Optimization:** Inlines methods, removes dead branches, constant propagation.
4. **DEX compilation:** Converts .class files to .dex (ART bytecode).

```
ProGuard rules file: android/app/proguard-rules.pro

# Keep React Native classes (never strip these)
-keep class com.facebook.react.** { *; }
-keep class com.facebook.hermes.** { *; }

# Keep your own app entry points
-keep class com.yourapp.MainActivity { *; }

# Keep JNI methods (native methods must not be renamed!)
-keepclasseswithmembernames class * {
    native <methods>;
}

# Keep Serializable classes
-keepclassmembers class * implements java.io.Serializable {
    static final long serialVersionUID;
    private void writeObject(java.io.ObjectOutputStream);
    private void readObject(java.io.ObjectInputStream);
}
```

**Diagnosing R8 issues:**
```bash
# Check what R8 removed (in build output):
cat android/app/build/outputs/mapping/release/usage.txt | grep "YourClass"

# Check final class name mappings:
cat android/app/build/outputs/mapping/release/mapping.txt
```

---

## 6. CMake — C++ Build System

React Native's C++ components are compiled via CMake:

```cmake
# android/app/CMakeLists.txt
cmake_minimum_required(VERSION 3.22.1)
project(myapp)

# Include React Native's CMake utilities
include(${REACT_ANDROID_DIR}/cmake-utils/ReactNative-application.cmake)

# Your C++ source files
add_library(
    myapp_jni             # library name (becomes libmyapp_jni.so)
    SHARED
    src/main/cpp/OnLoad.cpp        # JNI entry point
    src/main/cpp/NativeHasher.cpp
)

# Link against React Native and system libraries
target_link_libraries(
    myapp_jni
    react_nativemodule_core       # React Native C++ core
    jsi                           # JSI library
    react_render_core             # Fabric renderer core
    log                           # Android logging (ALOGD, ALOGE, etc.)
    android                       # Android native API
    ${CMAKE_DL_LIBS}              # Dynamic linking (dlopen)
)

# Include directories
target_include_directories(myapp_jni PRIVATE
    src/main/cpp
    ${REACT_ANDROID_DIR}/ReactCommon
    ${REACT_ANDROID_DIR}/ReactCommon/jsi
)
```

---

## 7. ABI Filters and App Size

React Native ships pre-compiled `.so` files for each CPU architecture:
- `arm64-v8a` — 64-bit ARM (most modern Android phones)
- `armeabi-v7a` — 32-bit ARM (older phones, required for broad compatibility)
- `x86` — 32-bit x86 (Android emulators)
- `x86_64` — 64-bit x86 (modern Android emulators)

```groovy
// For production, typically:
ndk { abiFilters "arm64-v8a", "armeabi-v7a" }
// Saves ~15-20MB vs including x86 variants
```

**Using Android App Bundle (AAB) — recommended:**
AAB lets Google Play serve the correct ABI to each device automatically. You don't need to bundle all ABIs in a single file — Google Play splits them.

```bash
# Build AAB (not APK) for Play Store submission
./gradlew bundleRelease
# Output: android/app/build/outputs/bundle/release/app-release.aab
```

---

## 8. Build Variants (Flavors)

Use product flavors for different build configurations (staging, production, white-label apps):

```groovy
android {
    flavorDimensions "environment"
    
    productFlavors {
        staging {
            dimension "environment"
            applicationIdSuffix ".staging"
            versionNameSuffix "-staging"
            buildConfigField "String", "API_BASE_URL", '"https://staging-api.example.com"'
        }
        production {
            dimension "environment"
            buildConfigField "String", "API_BASE_URL", '"https://api.example.com"'
        }
    }
}
```

```bash
./gradlew assembleStagingDebug    # staging + debug
./gradlew assembleProductionRelease  # production + release
```

---

## 9. AndroidManifest.xml — Critical React Native Settings

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    
    <!-- Required permissions -->
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
    
    <application
        android:name=".MainApplication"
        android:label="@string/app_name"
        android:icon="@mipmap/ic_launcher"
        android:allowBackup="false"
        android:theme="@style/AppTheme"
        android:largeHeap="true"        <!-- ← request larger heap for memory-intensive apps -->
        android:hardwareAccelerated="true">  <!-- ← enable GPU rendering (default true for API 14+) -->
        
        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:windowSoftInputMode="adjustResize"  <!-- keyboard resize behavior -->
            android:configChanges="keyboard|keyboardHidden|orientation|screenLayout|screenSize|smallestScreenSize|uiMode">
            <!-- configChanges: prevents Activity recreation on these changes (RN handles them) -->
            
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

---

## 10. Build Performance Optimization

```bash
# Run only the tasks you need (avoid full build)
./gradlew :app:assembleDebug   # only build debug APK

# Profile the build
./gradlew :app:assembleRelease --profile
# Generates: android/build/reports/profile/profile-*.html

# Parallel execution (already in gradle.properties but force it)
./gradlew :app:assembleRelease --parallel

# Build cache
./gradlew :app:assembleRelease --build-cache

# Use configuration cache (Gradle 8+)
./gradlew :app:assembleRelease --configuration-cache
```

**Key Build Time Contributors (and fixes):**

| Slow Step | Fix |
|-----------|-----|
| Annotation processing | Use KSP instead of KAPT |
| R8 full mode | Disable in debug builds |
| All ABI compilation | Use `abiFilters` in debug to only build one ABI |
| Metro JS bundle | Use `--active-arch-only` in Expo, run Metro separately |
| Cold Gradle daemon | Keep daemon alive: `org.gradle.daemon=true` |
