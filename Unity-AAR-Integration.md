# Unity Library Integration Guide

This guide explains how to integrate the Unity Library AAR files into your existing Android project.

## Prerequisites

- Android Studio (latest version recommended)
- Minimum SDK: 26
- Target SDK: 36
- Compile SDK: 36
- Java 11 or higher
- Gradle 8.13+
- Android Gradle Plugin 8.10.0+

## Important Notes

**CRITICAL:** You must add `androidx.games:games-activity:3.0.5` dependency to your build.gradle, otherwise the app will crash with `NoClassDefFoundError: com/google/androidgamesdk/GameActivity`

## Table of Contents

1. [Building the AAR Files](#building-the-aar-files)
2. [Setting Up Your Android Project](#setting-up-your-android-project)
3. [Integrating Unity Library](#integrating-unity-library)
4. [Launching Unity from Your App](#launching-unity-from-your-app)
5. [Communication Between Unity and Android](#communication-between-unity-and-android)
6. [Troubleshooting](#troubleshooting)

---

## 1. Building the AAR Files

### Step 1.1: Build Unity Library AAR

From the Unity exported Android project directory:

```bash
cd /path/to/AndroidExport-SDK
./gradlew :unityLibrary:assembleRelease
```

This generates:
- `unityLibrary/build/outputs/aar/unityLibrary-release.aar`

### Step 1.2: Build XR Manifest AAR

```bash
./gradlew :unityLibrary:xrmanifest.androidlib:assembleRelease
```

This generates:
- `unityLibrary/xrmanifest.androidlib/build/outputs/aar/xrmanifest.androidlib-release.aar`

### Step 1.3: Collect All Required Files

The following files are needed for integration:

**AAR Files:**
- `unityLibrary/build/outputs/aar/unityLibrary-release.aar` (38MB)
- `unityLibrary/xrmanifest.androidlib/build/outputs/aar/xrmanifest.androidlib-release.aar`
- `unityLibrary/libs/ARPresto.aar`
- `unityLibrary/libs/UnityARCore.aar`
- `unityLibrary/libs/arcore_client.aar`
- `unityLibrary/libs/unityandroidpermissions.aar`

---

## 2. Setting Up Your Android Project

### Step 2.1: Create or Open Your Android Project

Open Android Studio and create a new project or open your existing project.

### Step 2.2: Create libs Directory

In your app module, create a `libs` directory if it doesn't exist:

```bash
mkdir -p app/libs
```

### Step 2.3: Copy All AAR and JAR Files

Copy all the files collected in Step 1.3 to your `app/libs` directory:

---

## 3. Integrating Unity Library

### Step 3.1: Update Project-level build.gradle

File: `build.gradle` (Project level)

```gradle
plugins {
    id 'com.android.application' version '8.10.0' apply false
    id 'com.android.library' version '8.10.0' apply false
}
```

### Step 3.2: Update settings.gradle

File: `settings.gradle` (Project level)

```gradle
pluginManagement {
    repositories {
        gradlePluginPortal()
        google()
        mavenCentral()
    }
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.PREFER_SETTINGS)

    repositories {
        google()
        mavenCentral()

        // Add flatDir to resolve local AAR files
         flatDir {
            dirs file("app/libs")
        }
    }
}

rootProject.name = "Your App Name"
include ':app'
```

### Step 3.3: Update App-level build.gradle

File: `app/build.gradle`

**TESTED AND WORKING CONFIGURATION:**

```gradle
plugins {
    alias(libs.plugins.android.application)
}

android {
    namespace 'com.yourcompany.yourapp'
    compileSdk 36

    defaultConfig {
        applicationId "com.yourcompany.yourapp"
        minSdk 26
        targetSdk 36
        versionCode 1
        versionName "1.0"

        // Unity requires arm64-v8a architecture
        ndk {
            abiFilters "arm64-v8a"
        }

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }

    compileOptions {
        sourceCompatibility JavaVersion.VERSION_11
        targetCompatibility JavaVersion.VERSION_11
    }

    packaging {
        jniLibs {
            useLegacyPackaging true
        }
    }

    // Important: Prevent duplicate files error
    packagingOptions {
        pickFirst 'lib/arm64-v8a/libc++_shared.so'
        pickFirst 'lib/arm64-v8a/libunity.so'
        pickFirst 'lib/arm64-v8a/libil2cpp.so'
        pickFirst 'lib/arm64-v8a/libmain.so'
    }
}

dependencies {
    // Your existing AndroidX dependencies (adjust as needed)
    implementation 'androidx.appcompat:appcompat:1.6.1'
    implementation 'com.google.android.material:material:1.11.0'
    implementation 'androidx.activity:activity:1.8.0'
    implementation 'androidx.constraintlayout:constraintlayout:2.1.4'

    // Testing dependencies
    testImplementation 'junit:junit:4.13.2'
    androidTestImplementation 'androidx.test.ext:junit:1.1.5'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.5.1'

    // Unity Library - Main AAR files
    implementation(name: 'unityLibrary-release', ext: 'aar')
    implementation(name: 'xrmanifest.androidlib-release', ext: 'aar')

    // Unity Dependencies - Additional AAR files
    implementation(name: 'arcore_client', ext: 'aar')
    implementation(name: 'UnityARCore', ext: 'aar')
    implementation(name: 'ARPresto', ext: 'aar')
    implementation(name: 'unityandroidpermissions', ext: 'aar')

    // ⚠️ CRITICAL: This dependency is REQUIRED to prevent crash
    // Without this, you'll get: NoClassDefFoundError: com/google/androidgamesdk/GameActivity
    implementation 'androidx.games:games-activity:3.0.5'
}
```

**Key Points:**
- ✅ `androidx.games:games-activity:3.0.5` is **MANDATORY** - app will crash without it
- ✅ Java 11 is sufficient (Java 17 also works)
- ✅ `ndk { abiFilters "arm64-v8a" }` matches Unity's build
- ✅ `useLegacyPackaging true` is required for native libraries
- ✅ `packagingOptions` prevents duplicate library conflicts

### Step 3.4: Update AndroidManifest.xml

File: `app/src/main/AndroidManifest.xml`

Add necessary permissions and Unity activity declaration:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <!-- Unity Required Permissions -->
    <uses-permission android:name="android.permission.INTERNET" />

    <!-- ARCore Permissions (if using AR features) -->
    <uses-permission android:name="android.permission.CAMERA" />

    <!-- Required for Unity - OpenGL ES 3.0 support -->
    <uses-feature
        android:glEsVersion="0x00030000"
        android:required="true" />
    <uses-feature
        android:name="android.hardware.touchscreen"
        android:required="false" />
    <uses-feature
        android:name="android.hardware.touchscreen.multitouch"
        android:required="false" />
    <uses-feature
        android:name="android.hardware.touchscreen.multitouch.distinct"
        android:required="false" />
    <uses-feature
        android:name="android.hardware.camera"
        android:required="false" />

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.YourApp"
        android:hardwareAccelerated="true"
        android:extractNativeLibs="true">

        <!-- Your Main Activity -->
        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:theme="@style/Theme.YourApp">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

    </application>

</manifest>
```

**Important Notes:**
- ✅ `android:extractNativeLibs="true"` is required for Unity native libraries
- ✅ `android:hardwareAccelerated="true"` enables hardware acceleration
- ✅ `UnityPlayerGameActivity` must be declared (even if launching from your own activity)
- ✅ OpenGL ES 3.0 is required for Unity rendering

---

## 4. Launching Unity from Your App

### Method 1: Launch Unity in a Separate Activity

This is the recommended approach for most use cases.

#### Java Implementation

File: `app/src/main/java/com/yourcompany/yourapp/MainActivity.java`

```java
package com.yourcompany.yourapp;

import android.content.Intent;
import android.os.Bundle;
import android.widget.Button;
import androidx.appcompat.app.AppCompatActivity;
import com.unity3d.player.UnityPlayerGameActivity;

public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        findViewById(R.id.launchARButton).setOnClickListener(v -> launchARActivity());

    }

    private void launchARActivity() {
        // Create intent to launch Unity
        Intent intent = new Intent(this, UnityPlayerGameActivity.class);
        startActivity(intent);
    }
}
```


#### Layout File

File: `app/src/main/res/layout/activity_main.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">


    <Button
        android:id="@+id/launchARButton"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_centerInParent="true"
        android:layout_margin="54dp"
        android:text="Launch AR Activity" />

</RelativeLayout>
```

---

## 5. Communication Between Unity and Android

### Sending Messages from Android to Unity

You can call Unity methods from your Android code:

```java
import com.unity3d.player.UnityPlayer;

// Call a method on a Unity GameObject
UnityPlayer.UnitySendMessage("GameObjectName", "MethodName", "messageParameter");

// Example: Send score to Unity
public void sendScoreToUnity(int score) {
    UnityPlayer.UnitySendMessage("GameManager", "OnScoreReceived", String.valueOf(score));
}
```

In your Unity C# script:

```csharp
// Attach this to a GameObject named "GameManager"
public class GameManager : MonoBehaviour
{
    public void OnScoreReceived(string score)
    {
        Debug.Log("Received score from Android: " + score);
        int scoreValue = int.Parse(score);
        // Process the score
    }
}
```

### Sending Messages from Unity to Android

From Unity C# script, you can call Android methods:

```csharp
using UnityEngine;

public class AndroidCommunication : MonoBehaviour
{
    private AndroidJavaObject currentActivity;

    void Start()
    {
        #if UNITY_ANDROID && !UNITY_EDITOR
        using (AndroidJavaClass unityPlayer = new AndroidJavaClass("com.unity3d.player.UnityPlayer"))
        {
            currentActivity = unityPlayer.GetStatic<AndroidJavaObject>("currentActivity");
        }
        #endif
    }

    public void CallAndroidMethod()
    {
        #if UNITY_ANDROID && !UNITY_EDITOR
        if (currentActivity != null)
        {
            currentActivity.Call("onUnityCallback", "Hello from Unity!");
        }
        #endif
    }
}
```

In your MainActivity:

```java
public void onUnityCallback(String message) {
    runOnUiThread(() -> {
        Toast.makeText(this, "Unity says: " + message, Toast.LENGTH_SHORT).show();
    });
}
```

---

## 6. Troubleshooting

### Issue 1: GameActivity ClassNotFoundException (MOST COMMON)

**Error:**
```
java.lang.NoClassDefFoundError: Failed resolution of: Lcom/google/androidgamesdk/GameActivity;
Caused by: java.lang.ClassNotFoundException: Didn't find class "com.google.androidgamesdk.GameActivity"
```

**Root Cause:** Missing `androidx.games:games-activity` dependency

**Solution:** Add this to your `app/build.gradle` dependencies:
```gradle
implementation 'androidx.games:games-activity:3.0.5'
```

**This is MANDATORY** - UnityPlayerGameActivity extends GameActivity from this library.

### Issue 2: AAR Files Not Found

**Error:** `Could not resolve all dependencies` or `Failed to resolve: unityLibrary-release`

**Solution:**
1. Verify AAR files are in `app/libs/` directory
2. Check `settings.gradle` has flatDir configured:
```gradle
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        flatDir {
            dirs file("app/libs")
        }
    }
}
```
3. Sync Gradle and rebuild: `./gradlew clean build --refresh-dependencies`

### Issue 3: Native Library Not Found

**Error:** `java.lang.UnsatisfiedLinkError: dlopen failed: library "libmain.so" not found`

**Solution:**
1. Ensure `android:extractNativeLibs="true"` in AndroidManifest.xml `<application>` tag
2. Add to `app/build.gradle`:
```gradle
android {
    packaging {
        jniLibs {
            useLegacyPackaging true
        }
    }
}
```

### Issue 4: Duplicate Native Libraries

**Error:** `More than one file was found with OS independent path 'lib/arm64-v8a/libc++_shared.so'`

**Solution:** Add to `app/build.gradle`:
```gradle
android {
    packagingOptions {
        pickFirst 'lib/arm64-v8a/libc++_shared.so'
        pickFirst 'lib/arm64-v8a/libunity.so'
        pickFirst 'lib/arm64-v8a/libil2cpp.so'
        pickFirst 'lib/arm64-v8a/libmain.so'
    }
}
```

### Issue 5: Black Screen

**Error:** Unity launches but shows only a black screen

**Solution:**
1. Verify OpenGL ES 3.0 is declared in AndroidManifest.xml:
```xml
<uses-feature android:glEsVersion="0x00030000" android:required="true" />
```
2. Ensure `android:hardwareAccelerated="true"` in `<application>` tag
3. Check device supports OpenGL ES 3.0 (most devices from 2014+ do)

### Issue 6: App Crashes Immediately on Launch

**Possible Causes:**
1. **Missing GameActivity dependency** - See Issue 1
2. **Missing native libraries** - See Issue 3
3. **Wrong ABI** - Ensure device is arm64-v8a compatible
4. **Missing permissions** - Check logcat for permission errors

**Debug Steps:**
```bash
# Check logcat for detailed error
adb logcat -s Unity:V AndroidRuntime:E

# Verify native libraries are in APK
cd app/build/outputs/apk/debug
unzip -l app-debug.apk | grep libmain.so
```

### Issue 7: Gradle Sync Fails

**Solution:**
1. Invalidate caches: **File → Invalidate Caches → Invalidate and Restart**
2. Clean project: `./gradlew clean`
3. Delete `.gradle` folder and rebuild
4. Ensure Gradle version is 8.13 or higher

---

## 7. Quick Reference Commands

### Building Unity AAR Files
```bash
# Navigate to Unity exported project
cd /path/to/AndroidExport-SDK

# Build main Unity library
./gradlew :unityLibrary:assembleRelease

# Build XR manifest library
./gradlew :unityLibrary:xrmanifest.androidlib:assembleRelease

# Check generated files
ls -lh unityLibrary/build/outputs/aar/
```

### Android Project Commands
```bash
# Clean project
./gradlew clean

# Build debug APK
./gradlew assembleDebug

# Install on device
./gradlew installDebug

# Build and install
./gradlew clean assembleDebug installDebug

# Check dependencies
./gradlew :app:dependencies

# View Unity logs
adb logcat -s Unity
```

---

## 8. Summary Checklist

### Phase 1: Building AAR Files
- [ ] Navigate to Unity exported Android project directory
- [ ] Run `./gradlew :unityLibrary:assembleRelease`
- [ ] Run `./gradlew :unityLibrary:xrmanifest.androidlib:assembleRelease`
- [ ] Verify `unityLibrary-release.aar` is generated
- [ ] Verify `xrmanifest.androidlib-release.aar` is generated

### Phase 2: Copying Files
- [ ] Create `app/libs` directory in your Android project
- [ ] Copy `unityLibrary-release.aar` to `app/libs/`
- [ ] Copy `xrmanifest.androidlib-release.aar` to `app/libs/`
- [ ] Copy all AAR files from `unityLibrary/libs/` to `app/libs/`

### Phase 3: Gradle Configuration
- [ ] Update `settings.gradle`:
  - [ ] Add `flatDir { dirs file("app/libs") }` in repositories
  - [ ] Ensure `google()` and `mavenCentral()` are present
- [ ] Update `app/build.gradle`:
  - [ ] Set `compileSdk 36`
  - [ ] Set `minSdk 26`, `targetSdk 36`
  - [ ] Add `ndk { abiFilters "arm64-v8a" }`
  - [ ] Set Java compatibility to VERSION_11 or higher
  - [ ] Add all Unity AAR dependencies
  - [ ] **Add `implementation 'androidx.games:games-activity:3.0.5'`** (CRITICAL!)
  - [ ] Add `packaging { jniLibs { useLegacyPackaging true } }`
  - [ ] Add `packagingOptions { pickFirst ... }` for duplicate files

### Phase 4: AndroidManifest Configuration
- [ ] Add Internet permission (if needed)
- [ ] Add Camera permission (if using AR)
- [ ] Add OpenGL ES 3.0 feature requirement
- [ ] Add touchscreen features (optional)
- [ ] Set `android:extractNativeLibs="true"` in `<application>`
- [ ] Set `android:hardwareAccelerated="true"` in `<application>`
- [ ] Declare `UnityPlayerGameActivity` activity

### Phase 5: Implementation
- [ ] Create MainActivity to launch Unity
- [ ] Add button or trigger to start UnityPlayerGameActivity
- [ ] Test launching Unity from your app

### Phase 6: Testing
- [ ] Sync Gradle files successfully
- [ ] Build project without errors
- [ ] Install on arm64-v8a device
- [ ] Launch app and verify MainActivity works
- [ ] Click launch button and verify Unity starts
- [ ] Test Unity functionality
- [ ] Test communication between Android and Unity (if applicable)

---

## Additional Resources

- [Unity as a Library Documentation](https://docs.unity3d.com/Manual/UnityasaLibrary.html)
- [Android Gradle Plugin Release Notes](https://developer.android.com/studio/releases/gradle-plugin)
- [AndroidX Games Activity](https://developer.android.com/jetpack/androidx/releases/games)
- [Unity Native Plugin Interface](https://docs.unity3d.com/Manual/NativePlugins.html)

---

## Document Information

**Version:** 2.0 (Updated with tested configuration)
**Last Updated:** November 2024
**Unity Version:** 6000.0.61f1
**Android API Level:** 36
**Minimum SDK:** 26
**Tested Configuration:** ✅ Working

**Critical Dependencies:**
- `androidx.games:games-activity:3.0.5` - **MANDATORY**
- All Unity AAR files must be in `app/libs/`
- Java 11+ required

