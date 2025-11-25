# Unity-Android Bridge Integration Guide (Kotlin)

Complete guide for integrating Unity as an Android library (.AAR) with two-way communication using Kotlin.

---

## Table of Contents

1. [Overview](#overview)
2. [How UnityAndroidBridge Works](#how-unityandroidbridge-works)
3. [Two-Way Communication Architecture](#two-way-communication-architecture)
4. [Unity Project Export](#unity-project-export)
5. [Building the AAR Library](#building-the-aar-library)
6. [Android Integration with Kotlin](#android-integration-with-kotlin)
7. [Complete Kotlin Implementation](#complete-kotlin-implementation)

---

## Overview

This integration enables:
- **Unity as a library**: Export Unity scene as an Android library (.AAR)
- **Android → Unity**: Pass data (MapCode, configurations) from Android to Unity
- **Unity → Android**: Return results when Unity closes
- **Back button handling**: Automatic data return on back navigation

### Communication Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    Native Android App                       │
│  • User enters MapCode: "MAP_ABC123"                        │
│  • Clicks "Open Navigation" button                          │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ Intent with extras:
                         │ MAP_CODE = "MAP_ABC123"
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                   Unity Scene (from .AAR)                   │
│  • UnityAndroidBridge receives Intent extras                │
│  • Reads MAP_CODE from Intent                               │
│  • Applies MapCode to MapLocalizationManager                │
│  • User navigates in Unity scene                            │
│  • User presses back button                                 │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ Result Intent:
                         │ RESULT_DATA = "MAP_ABC123"
                         │ MESSAGE = "Callback from Navigation"
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    Native Android App                       │
│  • onActivityResult() receives data                         │
│  • Processes returned MapCode                               │
│  • Updates UI with results                                  │
└─────────────────────────────────────────────────────────────┘
```

---

## How UnityAndroidBridge Works

The `UnityAndroidBridge.cs` component provides seamless two-way communication between Unity and Android.

### Core Components

#### 1. **Singleton Pattern**
```csharp
private static UnityAndroidBridge instance;

void Awake()
{
    if (instance == null)
    {
        instance = this;
        DontDestroyOnLoad(gameObject);
        InitializeAndroidBridge();
    }
}
```
- Ensures only one bridge instance exists
- Persists across scene loads

#### 2. **Android Activity Reference**
```csharp
AndroidJavaClass unityPlayer = new AndroidJavaClass("com.unity3d.player.UnityPlayer");
currentActivity = unityPlayer.GetStatic<AndroidJavaObject>("currentActivity");
```
- Gets reference to the Android Activity hosting Unity
- Required for reading Intent data and returning results

#### 3. **MapLocalizationManager Integration**
```csharp
mapLocalizationManager = FindFirstObjectByType<MapLocalizationManager>();
```
- Automatically finds the MapLocalizationManager in the scene
- Applies received MapCode to this manager

---

## Two-Way Communication Architecture

### Android → Unity Communication

#### Method 1: Intent Extras (Automatic - Recommended)
When launching Unity Activity with Intent extras:

**Android Side:**
```kotlin
val intent = Intent(this, UnityPlayerGameActivity::class.java)
intent.putExtra("MAP_CODE", "MAP_ABC123")
startActivityForResult(intent, UNITY_REQUEST_CODE)
```

**Unity Side (Automatic):**
```csharp
private void ReadMapCodeFromIntent()
{
    AndroidJavaObject intent = currentActivity.Call<AndroidJavaObject>("getIntent");
    AndroidJavaObject extras = intent.Call<AndroidJavaObject>("getExtras");
    string mapCode = extras.Call<string>("getString", "MAP_CODE");

    // Automatically applies to MapLocalizationManager
    ApplyMapCode(mapCode);
}
```

**What Happens:**
1. Bridge reads Intent extras on initialization
2. Extracts "MAP_CODE" value
3. Automatically applies to `mapLocalizationManager.mapOrMapsetCode`
4. No additional Unity code needed!

#### Method 2: UnitySendMessage (Runtime)
Send data to Unity while it's running:

**Android Side:**
```kotlin
// Option A: Simple key-value
UnityPlayer.UnitySendMessage("UnityAndroidBridge", "SetMapCode", "NEW_MAP_CODE")

// Option B: Key-value pairs
UnityPlayer.UnitySendMessage("UnityAndroidBridge", "ReceiveData", "mapcode:NEW_MAP_CODE")

// Option C: JSON data
val jsonData = """{"messageType": "mapcode", "payload": "NEW_MAP_CODE"}"""
UnityPlayer.UnitySendMessage("UnityAndroidBridge", "ReceiveJsonData", jsonData)
```

**Unity Side:**
```csharp
// Automatically handled by these methods:
public void SetMapCode(string mapCode) { }
public void ReceiveData(string data) { }
public void ReceiveJsonData(string jsonData) { }
```

### Unity → Android Communication

#### Returning Data on Exit

**Unity Side:**
```csharp
public void OnBackButtonPressed()
{
    string currentMapCode = GetCurrentMapCode();
    ReturnToAndroid(currentMapCode, "Callback from Navigation");
}

public void ReturnToAndroid(string resultData, string message = "")
{
    // Create result Intent
    AndroidJavaObject resultIntent = new AndroidJavaObject("android.content.Intent");
    resultIntent.Call<AndroidJavaObject>("putExtra", "RESULT_DATA", resultData);
    resultIntent.Call<AndroidJavaObject>("putExtra", "MESSAGE", message);

    // Set result and finish
    int RESULT_OK = -1;
    activity.Call("setResult", RESULT_OK, resultIntent);
    activity.Call("finish");
}
```

**Android Side:**
```kotlin
override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
    if (requestCode == UNITY_REQUEST_CODE && resultCode == RESULT_OK) {
        val resultData = data?.getStringExtra("RESULT_DATA")
        val message = data?.getStringExtra("MESSAGE")
        // Process the data
    }
}
```

---

## Unity Project Export

### Step 1: Configure Unity Build Settings

1. Open Unity Editor
2. Go to **File → Build Settings**
3. Select **Android** platform
4. Click **Switch Platform** (if needed)

### Step 2: Configure Player Settings

Click **Player Settings** and configure:

#### Package Information
- **Company Name**: `com.yourcompany`
- **Product Name**: `YourAppName`
- **Package Name**: `com.yourcompany.yourapp` *(Important: Note this)*

#### Android Settings
- **Minimum API Level**: 24 or higher
- **Scripting Backend**: IL2CPP (recommended)
- **Target Architectures**: ARM64 (required), ARMv7 (optional)

### Step 3: Export Project

1. In Build Settings, **check "Export Project"**
2. **Uncheck "Build App Bundle"** if checked
3. Click **Export**
4. Choose export location (e.g., `UnityExport`)

**Exported Structure:**
```
UnityExport/
├── launcher/
├── unityLibrary/     ← Contains Unity project
├── build.gradle
└── gradle.properties
```

---

## Building the AAR Library

### Step 1: Open in Android Studio

1. Open Android Studio
2. **File → Open** → Select `UnityExport` folder
3. Wait for Gradle sync

### Step 2: Build AAR

Open Terminal in Android Studio and run:

```bash
./gradlew :unityLibrary:assembleRelease
```

**Output Location:**
```
UnityExport/unityLibrary/build/outputs/aar/unityLibrary-release.aar
```

---

## Android Integration with Kotlin

### Step 1: Add AAR Files to Your Android Project

1. Create `libs` folder in your app module:
   ```
   YourAndroidApp/
   └── app/
       ├── libs/
       │   ├── unityLibrary-debug.aar          (or unityLibrary-release.aar)
       │   ├── xrmanifest.androidlib-release.aar
       │   ├── arcore_client.aar
       │   ├── UnityARCore.aar
       │   ├── ARPresto.aar
       │   └── unityandroidpermissions.aar
       └── build.gradle.kts
   ```

2. Copy all AAR files from the Unity export to `app/libs/`:
   - **unityLibrary-debug.aar** (or release): Main Unity library
   - **xrmanifest.androidlib-release.aar**: XR manifest support
   - **arcore_client.aar**: ARCore client library
   - **UnityARCore.aar**: Unity ARCore integration
   - **ARPresto.aar**: AR Presto library
   - **unityandroidpermissions.aar**: Unity Android permissions handler

### Step 2: Update app/build.gradle.kts

```kotlin
import org.gradle.kotlin.dsl.implementation

plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
}

android {
    namespace = "com.yourcompany.mainapp"
    compileSdk = 36

    defaultConfig {
        applicationId = "com.yourcompany.mainapp"
        minSdk = 28
        targetSdk = 36
        versionCode = 1
        versionName = "1.0"

        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            isMinifyEnabled = false
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_11
        targetCompatibility = JavaVersion.VERSION_11
    }
    kotlinOptions {
        jvmTarget = "11"
    }
}

dependencies {
    implementation(libs.androidx.core.ktx)
    implementation(libs.androidx.appcompat)
    implementation(libs.material)
    implementation(libs.androidx.activity)
    implementation(libs.androidx.constraintlayout)
    testImplementation(libs.junit)
    androidTestImplementation(libs.androidx.junit)
    androidTestImplementation(libs.androidx.espresso.core)

    // Unity Library (main)
    implementation(files("libs/unityLibrary-debug.aar"))
    // OR for release: implementation(files("libs/unityLibrary-release.aar"))

    // XR Manifest support
    implementation(files("libs/xrmanifest.androidlib-release.aar"))

    // ARCore dependencies
    implementation(files("libs/arcore_client.aar"))
    implementation(files("libs/UnityARCore.aar"))
    implementation(files("libs/ARPresto.aar"))

    // Unity Android Permissions
    implementation(files("libs/unityandroidpermissions.aar"))

    // Required: Games Activity for UnityPlayerGameActivity
    implementation("androidx.games:games-activity:3.0.5")
}
```

### Step 3: Sync Gradle

Click **Sync Now** when prompted.

---

## Complete Kotlin Implementation

### Step 1: Update AndroidManifest.xml

Add Unity activity and required permissions:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <!-- Required Permissions -->
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.CAMERA" />

    <!-- Required Features -->
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
        android:dataExtractionRules="@xml/data_extraction_rules"
        android:fullBackupContent="@xml/backup_rules"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.YourApp">

        <!-- Main Activity -->
        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <!-- Unity Activity -->
        <activity
            android:name="com.unity3d.player.UnityPlayerGameActivity"
            android:configChanges="mcc|mnc|locale|touchscreen|keyboard|keyboardHidden|navigation|orientation|screenLayout|uiMode|screenSize|smallestScreenSize|fontScale|layoutDirection|density"
            android:excludeFromRecents="false"
            android:exported="false"
            android:hardwareAccelerated="true"
            android:label="Unity Navigation"
            android:launchMode="standard"
            android:noHistory="false"
            android:process=":Unity"
            android:screenOrientation="fullSensor"
            android:taskAffinity=""
            tools:replace="android:exported,android:screenOrientation,android:launchMode,android:hardwareAccelerated" />

    </application>

</manifest>
```

### Step 2: Create MainActivity.kt

```kotlin
package com.yourcompany.mainapp

import android.content.Intent
import android.os.Bundle
import android.widget.Button
import android.widget.EditText
import android.widget.Toast
import androidx.appcompat.app.AppCompatActivity
import androidx.core.view.ViewCompat
import androidx.core.view.WindowInsetsCompat
import com.unity3d.player.UnityPlayerGameActivity

class MainActivity : AppCompatActivity() {

    companion object {
        private const val UNITY_REQUEST_CODE = 1001
    }

    private lateinit var mapCodeInput: EditText

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        // Handle window insets for edge-to-edge display
        ViewCompat.setOnApplyWindowInsetsListener(findViewById(R.id.main)) { v, insets ->
            val systemBars = insets.getInsets(WindowInsetsCompat.Type.systemBars())
            v.setPadding(systemBars.left, systemBars.top, systemBars.right, systemBars.bottom)
            insets
        }

        mapCodeInput = findViewById(R.id.mapCodeInput)
        val openNavigationButton = findViewById<Button>(R.id.openNavigationButton)

        openNavigationButton.setOnClickListener {
            openUnityNavigation()
        }
    }

    /**
     * Launch Unity with MapCode parameter.
     * This is the primary way to pass data from Android to Unity.
     */
    private fun openUnityNavigation() {
        val mapCode = mapCodeInput.text.toString().trim()

        if (mapCode.isEmpty()) {
            Toast.makeText(this, "Please enter a MapCode", Toast.LENGTH_SHORT).show()
            return
        }

        // Create intent to launch Unity using UnityPlayerGameActivity
        val intent = Intent(this, UnityPlayerGameActivity::class.java).apply {
            // Pass MapCode to Unity via Intent extras
            // UnityAndroidBridge will automatically read this
            putExtra("MAP_CODE", mapCode)
        }

        // Launch Unity and expect a result
        startActivityForResult(intent, UNITY_REQUEST_CODE)
    }

    /**
     * Receive result from Unity when it closes.
     * Unity calls ReturnToAndroid() which sends data back here.
     */
    override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
        super.onActivityResult(requestCode, resultCode, data)

        if (requestCode == UNITY_REQUEST_CODE) {
            if (resultCode == RESULT_OK && data != null) {
                // Get data returned from Unity
                val resultData = data.getStringExtra("RESULT_DATA")
                val message = data.getStringExtra("MESSAGE")

                Toast.makeText(
                    this,
                    "Returned from Unity!\nData: $resultData\nMessage: $message",
                    Toast.LENGTH_LONG
                ).show()

                // Process the returned data
                handleUnityResult(resultData, message)
            } else {
                Toast.makeText(this, "Unity closed without result", Toast.LENGTH_SHORT).show()
            }
        }
    }

    /**
     * Handle data returned from Unity.
     * Implement your business logic here.
     */
    private fun handleUnityResult(resultData: String?, message: String?) {
        println("Unity Result - Data: $resultData, Message: $message")

        // Example: Update the input field with returned data
        resultData?.let {
            mapCodeInput.setText(it)
        }
    }
}
```

> **Note**: This uses `UnityPlayerGameActivity` instead of the deprecated `UnityPlayerActivity`. The `UnityPlayerGameActivity` is the modern approach for Unity 2021.2+ and requires the `androidx.games:games-activity` dependency.

### Step 3: Create Layout (res/layout/activity_main.xml)

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp"
    android:gravity="center">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Unity Navigation Demo"
        android:textSize="24sp"
        android:textStyle="bold"
        android:layout_marginBottom="32dp" />

    <EditText
        android:id="@+id/mapCodeInput"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="Enter MapCode (e.g., MAP_ABC123)"
        android:inputType="text"
        android:layout_marginBottom="16dp" />

    <Button
        android:id="@+id/openNavigationButton"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Open Navigation"
        android:textSize="18sp" />

</LinearLayout>
```

---

## Advanced Kotlin Examples

### Example 1: Send Runtime Messages to Unity

```kotlin
import com.unity3d.player.UnityPlayer

class MainActivity : Activity() {

    /**
     * Send a new MapCode to Unity while it's running
     */
    private fun updateMapCodeInRuntime(newMapCode: String) {
        // Method 1: Direct method call
        UnityPlayer.UnitySendMessage(
            "UnityAndroidBridge",  // GameObject name
            "SetMapCode",          // Method name
            newMapCode             // Parameter
        )

        // Method 2: Key-value format
        UnityPlayer.UnitySendMessage(
            "UnityAndroidBridge",
            "ReceiveData",
            "mapcode:$newMapCode"
        )

        // Method 3: JSON format
        val jsonData = """
            {
                "messageType": "mapcode",
                "payload": "$newMapCode"
            }
        """.trimIndent()

        UnityPlayer.UnitySendMessage(
            "UnityAndroidBridge",
            "ReceiveJsonData",
            jsonData
        )
    }

    /**
     * Request POI list from Unity
     */
    private fun requestPOIList() {
        UnityPlayer.UnitySendMessage(
            "UnityAndroidBridge",
            "GetPOIList",
            ""
        )
    }
}
```

### Example 2: Dynamic MapCode Selection

```kotlin
import android.widget.Spinner
import android.widget.ArrayAdapter

class MainActivity : Activity() {

    private lateinit var mapCodeSpinner: Spinner

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        setupMapCodeSpinner()
    }

    private fun setupMapCodeSpinner() {
        mapCodeSpinner = findViewById(R.id.mapCodeSpinner)

        // Define available map codes
        val mapCodes = arrayOf(
            "MAP_BUILDING_A",
            "MAP_BUILDING_B",
            "MAP_CAMPUS",
            "MAP_FLOOR_1",
            "MAP_FLOOR_2"
        )

        val adapter = ArrayAdapter(
            this,
            android.R.layout.simple_spinner_item,
            mapCodes
        )
        adapter.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item)
        mapCodeSpinner.adapter = adapter

        val openButton = findViewById<Button>(R.id.openNavigationButton)
        openButton.setOnClickListener {
            val selectedMapCode = mapCodeSpinner.selectedItem.toString()
            launchUnityWithMapCode(selectedMapCode)
        }
    }

    private fun launchUnityWithMapCode(mapCode: String) {
        val intent = Intent(this, UnityPlayerGameActivity::class.java).apply {
            putExtra("MAP_CODE", mapCode)
        }
        startActivityForResult(intent, UNITY_REQUEST_CODE)
    }
}
```

### Example 3: Handle Multiple Return Data Types

```kotlin
class MainActivity : Activity() {

    override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
        super.onActivityResult(requestCode, resultCode, data)

        if (requestCode == UNITY_REQUEST_CODE && resultCode == RESULT_OK && data != null) {

            // Standard return data
            val resultData = data.getStringExtra("RESULT_DATA")
            val message = data.getStringExtra("MESSAGE")

            // Check if we received POI list JSON
            if (resultData?.startsWith("{") == true) {
                handleJsonResult(resultData)
            } else {
                handleStandardResult(resultData, message)
            }
        }
    }

    private fun handleJsonResult(jsonData: String) {
        try {
            // Parse JSON (using your preferred JSON library)
            // Example with org.json:
            val jsonObject = JSONObject(jsonData)
            val pois = jsonObject.getJSONArray("pois")

            println("Received ${pois.length()} POIs from Unity")

            for (i in 0 until pois.length()) {
                val poi = pois.getJSONObject(i)
                val name = poi.getString("name")
                println("POI: $name")
            }

        } catch (e: Exception) {
            println("Error parsing JSON: ${e.message}")
        }
    }

    private fun handleStandardResult(resultData: String?, message: String?) {
        println("Standard result - Data: $resultData, Message: $message")

        // Update UI or perform actions based on result
        resultData?.let {
            // Save to preferences, update database, etc.
        }
    }
}
```

---

## What Changes Are Required on Android Kotlin Side

### Summary of Required Changes

#### 1. **AndroidManifest.xml**
- Add Unity activity declaration
- Add required permissions (INTERNET, CAMERA)
- Add required features (OpenGL ES, touchscreen)

#### 2. **build.gradle.kts**
- Add all Unity AAR dependencies:
  - `implementation(files("libs/unityLibrary-debug.aar"))` (or release)
  - `implementation(files("libs/xrmanifest.androidlib-release.aar"))`
  - `implementation(files("libs/arcore_client.aar"))`
  - `implementation(files("libs/UnityARCore.aar"))`
  - `implementation(files("libs/ARPresto.aar"))`
  - `implementation(files("libs/unityandroidpermissions.aar"))`
- Add games-activity dependency: `implementation("androidx.games:games-activity:3.0.5")`

#### 3. **MainActivity.kt**
- Create Intent to launch `UnityPlayerGameActivity`
- Pass MapCode via `intent.putExtra("MAP_CODE", mapCode)`
- Use `startActivityForResult()` to expect data back
- Override `onActivityResult()` to receive returned data
- Extract `RESULT_DATA` and `MESSAGE` from returned Intent

#### 4. **Layout XML**
- Create input field for MapCode
- Create button to launch Unity
- Add any UI elements to display returned data

#### 5. **No Additional Code Needed**
- UnityAndroidBridge handles all Unity-side logic automatically
- No need to modify Unity scene or write additional C# code
- Bridge automatically reads Intent extras and applies MapCode

---

## Key Integration Points

### Android → Unity (Automatic)
```kotlin
// In Android Kotlin
val intent = Intent(this, UnityPlayerGameActivity::class.java)
intent.putExtra("MAP_CODE", "YOUR_MAP_CODE")  // ← Android sends
startActivityForResult(intent, REQUEST_CODE)
```

```csharp
// In Unity C# (Automatic)
private void ReadMapCodeFromIntent()
{
    // UnityAndroidBridge automatically:
    // 1. Reads Intent extras
    // 2. Extracts MAP_CODE
    // 3. Applies to mapLocalizationManager.mapOrMapsetCode
}
```

### Unity → Android (On Back Button)
```csharp
// In Unity C# (Automatic on back button)
public void OnBackButtonPressed()
{
    string currentMapCode = GetCurrentMapCode();
    ReturnToAndroid(currentMapCode, "Callback from Navigation");  // ← Unity sends back
}
```

```kotlin
// In Android Kotlin
override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
    val resultData = data?.getStringExtra("RESULT_DATA")  // ← Android receives
    val message = data?.getStringExtra("MESSAGE")
}
```

---

## Complete Communication Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                         ANDROID KOTLIN                          │
│                                                                 │
│  MainActivity.kt:                                               │
│  val intent = Intent(this, UnityPlayerGameActivity::class.java)│
│  intent.putExtra("MAP_CODE", "MAP_ABC123")                     │
│  startActivityForResult(intent, 1001)                          │
│                                                                 │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ Intent Extras: MAP_CODE = "MAP_ABC123"
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      UNITY (UnityAndroidBridge)                 │
│                                                                 │
│  1. InitializeAndroidBridge()                                   │
│     - Gets Android Activity reference                           │
│                                                                 │
│  2. ReadMapCodeFromIntent()                                     │
│     - Reads Intent.getExtras()                                  │
│     - Extracts MAP_CODE = "MAP_ABC123"                          │
│                                                                 │
│  3. ApplyMapCode("MAP_ABC123")                                  │
│     - Sets mapLocalizationManager.mapOrMapsetCode               │
│                                                                 │
│  4. User navigates in Unity scene                               │
│                                                                 │
│  5. User presses back button                                    │
│                                                                 │
│  6. OnBackButtonPressed()                                       │
│     - Gets currentMapCode                                       │
│     - Calls ReturnToAndroid()                                   │
│                                                                 │
│  7. ReturnToAndroid(resultData, message)                        │
│     - Creates Intent with extras                                │
│     - Calls setResult(RESULT_OK, intent)                        │
│     - Calls activity.finish()                                   │
│                                                                 │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ Result Intent:
                             │ RESULT_DATA = "MAP_ABC123"
                             │ MESSAGE = "Callback from Navigation"
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                         ANDROID KOTLIN                          │
│                                                                 │
│  MainActivity.kt:                                               │
│  override fun onActivityResult(requestCode, resultCode, data) { │
│      val resultData = data?.getStringExtra("RESULT_DATA")      │
│      val message = data?.getStringExtra("MESSAGE")             │
│      handleUnityResult(resultData, message)                    │
│  }                                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Quick Start Checklist

- [ ] Export Unity project with "Export Project" enabled
- [ ] Build `unityLibrary-debug.aar` (or release) using Gradle
- [ ] Copy all AAR files to Android project's `app/libs/` folder:
  - [ ] `unityLibrary-debug.aar` (or release)
  - [ ] `xrmanifest.androidlib-release.aar`
  - [ ] `arcore_client.aar`
  - [ ] `UnityARCore.aar`
  - [ ] `ARPresto.aar`
  - [ ] `unityandroidpermissions.aar`
- [ ] Update `build.gradle.kts` with all AAR dependencies and games-activity
- [ ] Add Unity activity (`UnityPlayerGameActivity`) to `AndroidManifest.xml`
- [ ] Add required permissions and features to manifest
- [ ] Create `MainActivity.kt` extending `AppCompatActivity` with Intent launch code
- [ ] Implement `onActivityResult()` to receive data
- [ ] Create layout XML with MapCode input and button
- [ ] Build and run Android app
- [ ] Test: Enter MapCode → Open Unity → Press Back → Receive result

---

## Notes

- **UnityAndroidBridge** handles all Unity-side logic automatically
- **No additional Unity code** required for basic MapCode integration
- **Intent extras** are the recommended way to pass initial data
- **UnitySendMessage** can be used for runtime updates
- **Back button** automatically triggers data return
- **Result Intent** contains `RESULT_DATA` and `MESSAGE` keys
- **UnityPlayerGameActivity** is the modern Activity class (Unity 2021.2+), replacing deprecated `UnityPlayerActivity`
- **games-activity dependency** is required for `UnityPlayerGameActivity` to work
- **Multiple AAR files** are needed for AR functionality - don't forget the ARCore-related AARs

---

Generated for MultiSet SDK Unity-Android Integration
