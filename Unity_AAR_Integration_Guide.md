# Unity as Android Library (.AAR) - Complete Integration Guide

Complete solution for embedding Unity as a library (.AAR) in a native Android application with two-way communication.

---

## Table of Contents

1. [Overview](#overview)
2. [How UnityAndroidBridge Works](#how-unityandroidbridge-works)
3. [Two-Way Communication Architecture](#two-way-communication-architecture)
4. [Step 1: Export Unity Project](#step-1-export-unity-project)
5. [Step 2: Build AAR Library](#step-2-build-aar-library)
6. [Step 3: Integrate AAR into Android App](#step-3-integrate-aar-into-android-app)
7. [Step 4: Android Implementation (Java)](#step-4-android-implementation-java)
8. [Step 5: Android Implementation (Kotlin)](#step-5-android-implementation-kotlin)
9. [Advanced Usage Examples](#advanced-usage-examples)

---

## Overview

This package enables you to:
- Export Unity scene as Android library (.AAR) - [Follow this guide](https://github.com/shyam-barange/Unity-Android-Integration/blob/main/Unity-AAR-Integration.md)
- Integrate Unity into native Android app
- **Pass MapCode from Android → Unity** (automatic)
- **Return data from Unity → Android** (on back button)
- Handle back button navigation seamlessly

### What You Get

**UnityAndroidBridge** - A ready-to-use Unity component that:
- ✅ Automatically reads Intent extras when Unity launches
- ✅ Applies MapCode to MapLocalizationManager
- ✅ Handles back button and returns data to Android
- ✅ Supports runtime messaging between Android and Unity
- ✅ Requires zero additional Unity code for basic integration

### App Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    Native Android App                       │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  User enters MapCode: "MAP_ABC123"                   │  │
│  │  Clicks "Open Navigation" button                     │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ Intent with extras:
                         │ MAP_CODE = "MAP_ABC123"
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                   Unity Scene (from .AAR)                   │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  UnityAndroidBridge.cs (Automatic)                   │  │
│  │  • Reads MapCode from Intent extras                  │  │
│  │  • Sets mapLocalizationManager.mapOrMapsetCode       │  │
│  │  • User navigates in Unity scene                     │  │
│  │  • User presses back button                          │  │
│  │  • Returns data to Android                           │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ Result Intent:
                         │ RESULT_DATA = "MAP_ABC123"
                         │ MESSAGE = "Callback from Navigation"
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    Native Android App                       │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  onActivityResult() receives data                    │  │
│  │  Displays result to user                             │  │
│  │  Processes returned data                             │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## How UnityAndroidBridge Works

The `UnityAndroidBridge.cs` component is the core of the two-way communication system. Understanding how it works will help you integrate Unity seamlessly into your Android app.

### Core Architecture

#### 1. Singleton Pattern
```csharp
private static UnityAndroidBridge instance;

void Awake()
{
    if (instance == null)
    {
        instance = this;
        DontDestroyOnLoad(gameObject);  // Persists across scenes
        InitializeAndroidBridge();
    }
}
```
**Purpose:**
- Ensures only one bridge instance exists throughout the Unity lifecycle
- Persists across scene loads so communication never breaks
- Initializes Android connection automatically on startup

#### 2. Android Activity Reference
```csharp
void InitializeAndroidBridge()
{
    // Get reference to the Android Activity hosting Unity
    AndroidJavaClass unityPlayer = new AndroidJavaClass("com.unity3d.player.UnityPlayer");
    currentActivity = unityPlayer.GetStatic<AndroidJavaObject>("currentActivity");

    // Automatically read Intent extras
    ReadMapCodeFromIntent();
}
```
**Purpose:**
- Gets a reference to the Android Activity that launched Unity
- Required for reading Intent data and returning results
- Only works on Android devices (not in Unity Editor)

#### 3. Automatic Intent Reading (Android → Unity)
```csharp
private void ReadMapCodeFromIntent()
{
    // Get the Intent that launched Unity
    AndroidJavaObject intent = currentActivity.Call<AndroidJavaObject>("getIntent");
    AndroidJavaObject extras = intent.Call<AndroidJavaObject>("getExtras");

    // Extract MAP_CODE from Intent extras
    string mapCode = extras.Call<string>("getString", "MAP_CODE");

    if (!string.IsNullOrEmpty(mapCode))
    {
        receivedMapCode = mapCode;
        // Automatically apply if MapLocalizationManager is ready
        ApplyMapCode(mapCode);
    }
}
```
**Purpose:**
- Automatically reads data passed from Android via Intent extras
- Extracts the "MAP_CODE" parameter
- Applies it to MapLocalizationManager without any manual intervention

#### 4. MapLocalizationManager Integration
```csharp
void Start()
{
    // Find MapLocalizationManager in the scene
    mapLocalizationManager = FindFirstObjectByType<MapLocalizationManager>();

    // Apply MapCode if it was received before Start
    if (!string.IsNullOrEmpty(receivedMapCode))
    {
        ApplyMapCode(receivedMapCode);
    }
}

private void ApplyMapCode(string mapCode)
{
    if (mapLocalizationManager != null)
    {
        mapLocalizationManager.mapOrMapsetCode = mapCode;
        Debug.Log($"MapCode successfully applied: {mapCode}");
    }
}
```
**Purpose:**
- Automatically finds and connects to MapLocalizationManager
- Applies received MapCode to enable map functionality
- Works seamlessly without any additional setup

#### 5. Returning to Android (Unity → Android)
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

    // Set result and finish activity
    int RESULT_OK = -1; // Activity.RESULT_OK
    activity.Call("setResult", RESULT_OK, resultIntent);
    activity.Call("finish");
}
```
**Purpose:**
- Handles back button press automatically
- Creates an Intent with result data
- Returns data to Android and closes Unity activity
- Android receives data in `onActivityResult()`

### Key Features

#### ✅ Automatic Intent Reading
- Unity automatically reads `MAP_CODE` from Intent extras when launched
- No manual method calls needed from Android side
- Just pass data in Intent, bridge handles the rest

#### ✅ Automatic MapCode Application
- Bridge finds MapLocalizationManager automatically
- Applies MapCode without any additional code
- Updates `mapLocalizationManager.mapOrMapsetCode` field

#### ✅ Back Button Handling
- Intercepts back button press
- Returns current MapCode to Android
- Closes Unity gracefully

#### ✅ Runtime Messaging Support
- Supports `UnitySendMessage` for runtime updates
- Can receive data while Unity is running
- Supports simple strings, key-value pairs, and JSON

---

## Two-Way Communication Architecture

### Communication Method 1: Intent Extras (Recommended for Launch)

#### Android → Unity (Automatic)

**Android Side:**
```java
// Java
Intent intent = new Intent(this, UnityPlayerActivity.class);
intent.putExtra("MAP_CODE", "MAP_ABC123");
startActivityForResult(intent, UNITY_REQUEST_CODE);
```

```kotlin
// Kotlin
val intent = Intent(this, UnityPlayerActivity::class.java)
intent.putExtra("MAP_CODE", "MAP_ABC123")
startActivityForResult(intent, UNITY_REQUEST_CODE)
```

**Unity Side (Automatic - No Code Needed):**
```csharp
// This happens automatically in UnityAndroidBridge:
private void ReadMapCodeFromIntent()
{
    AndroidJavaObject intent = currentActivity.Call<AndroidJavaObject>("getIntent");
    AndroidJavaObject extras = intent.Call<AndroidJavaObject>("getExtras");
    string mapCode = extras.Call<string>("getString", "MAP_CODE");

    // Automatically applied to MapLocalizationManager
    ApplyMapCode(mapCode);
}
```

**Result:**
- MapCode is automatically read from Intent
- Automatically applied to `mapLocalizationManager.mapOrMapsetCode`
- No Unity code modifications needed!

---

### Communication Method 2: UnitySendMessage (Runtime Updates)

#### Android → Unity (While Unity is Running)

**Android Side:**
```java
// Java
// Option 1: Direct method call
UnityPlayer.UnitySendMessage("UnityAndroidBridge", "SetMapCode", "NEW_MAP_CODE");

// Option 2: Key-value format
UnityPlayer.UnitySendMessage("UnityAndroidBridge", "ReceiveData", "mapcode:NEW_MAP_CODE");

// Option 3: JSON format
String jsonData = "{\"messageType\": \"mapcode\", \"payload\": \"NEW_MAP_CODE\"}";
UnityPlayer.UnitySendMessage("UnityAndroidBridge", "ReceiveJsonData", jsonData);
```

```kotlin
// Kotlin
// Option 1: Direct method call
UnityPlayer.UnitySendMessage("UnityAndroidBridge", "SetMapCode", "NEW_MAP_CODE")

// Option 2: Key-value format
UnityPlayer.UnitySendMessage("UnityAndroidBridge", "ReceiveData", "mapcode:NEW_MAP_CODE")

// Option 3: JSON format
val jsonData = """{"messageType": "mapcode", "payload": "NEW_MAP_CODE"}"""
UnityPlayer.UnitySendMessage("UnityAndroidBridge", "ReceiveJsonData", jsonData)
```

**Unity Side (Automatic - Already Implemented):**
```csharp
// These methods are already implemented in UnityAndroidBridge:

public void SetMapCode(string mapCode)
{
    receivedMapCode = mapCode;
    ApplyMapCode(mapCode);
}

public void ReceiveData(string data)
{
    string[] parts = data.Split(':');
    string key = parts[0];
    string value = parts[1];
    HandleDataByKey(key, value);
}

public void ReceiveJsonData(string jsonData)
{
    BridgeData data = JsonUtility.FromJson<BridgeData>(jsonData);
    HandleBridgeData(data);
}
```

---

### Communication Method 3: Return to Android (On Exit)

#### Unity → Android (When Closing)

**Unity Side (Automatic - Already Implemented):**
```csharp
// Automatically called when back button is pressed:
public void OnBackButtonPressed()
{
    string currentMapCode = GetCurrentMapCode();
    ReturnToAndroid(currentMapCode, "Callback from Navigation");
}

// Can also be called manually from other scripts:
public void ReturnToAndroid(string resultData, string message = "")
{
    AndroidJavaObject resultIntent = new AndroidJavaObject("android.content.Intent");
    resultIntent.Call<AndroidJavaObject>("putExtra", "RESULT_DATA", resultData);
    resultIntent.Call<AndroidJavaObject>("putExtra", "MESSAGE", message);

    int RESULT_OK = -1;
    activity.Call("setResult", RESULT_OK, resultIntent);
    activity.Call("finish");
}
```

**Android Side (Required Implementation):**
```java
// Java
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    super.onActivityResult(requestCode, resultCode, data);

    if (requestCode == UNITY_REQUEST_CODE && resultCode == RESULT_OK && data != null) {
        String resultData = data.getStringExtra("RESULT_DATA");
        String message = data.getStringExtra("MESSAGE");

        // Process the returned data
        handleUnityResult(resultData, message);
    }
}
```

```kotlin
// Kotlin
override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
    super.onActivityResult(requestCode, resultCode, data)

    if (requestCode == UNITY_REQUEST_CODE && resultCode == RESULT_OK && data != null) {
        val resultData = data?.getStringExtra("RESULT_DATA")
        val message = data?.getStringExtra("MESSAGE")

        // Process the returned data
        handleUnityResult(resultData, message)
    }
}
```

---

### Complete Communication Flow Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                      ANDROID APPLICATION                     │
│                                                              │
│  MainActivity:                                               │
│  • User enters MapCode                                       │
│  • Creates Intent with MAP_CODE extra                        │
│  • Calls startActivityForResult()                            │
│                                                              │
└───────────────────────────┬──────────────────────────────────┘
                            │
                            │ 1. Launch with Intent
                            │    MAP_CODE = "MAP_ABC123"
                            │
                            ▼
┌──────────────────────────────────────────────────────────────┐
│                     UNITY (UnityAndroidBridge)               │
│                                                              │
│  Awake():                                                    │
│  └─> InitializeAndroidBridge()                              │
│      └─> Get Android Activity reference                     │
│      └─> ReadMapCodeFromIntent()                            │
│          └─> Extract "MAP_CODE" from Intent                 │
│          └─> Save to receivedMapCode                        │
│                                                              │
│  Start():                                                    │
│  └─> Find MapLocalizationManager                            │
│  └─> ApplyMapCode(receivedMapCode)                          │
│      └─> mapLocalizationManager.mapOrMapsetCode = mapCode   │
│                                                              │
│  ─────── Unity Scene Running ───────                         │
│                                                              │
│  Optional: Runtime Updates from Android                     │
│  └─> UnitySendMessage("UnityAndroidBridge", ...)            │
│      └─> SetMapCode() / ReceiveData() / ReceiveJsonData()   │
│                                                              │
│  ─────── User Presses Back Button ───────                    │
│                                                              │
│  OnBackButtonPressed():                                      │
│  └─> GetCurrentMapCode()                                    │
│  └─> ReturnToAndroid(mapCode, "Callback from Navigation")   │
│      └─> Create Intent with RESULT_DATA & MESSAGE           │
│      └─> Call setResult(RESULT_OK, intent)                  │
│      └─> Call finish()                                      │
│                                                              │
└───────────────────────────┬──────────────────────────────────┘
                            │
                            │ 2. Return with Result Intent
                            │    RESULT_DATA = "MAP_ABC123"
                            │    MESSAGE = "Callback from Navigation"
                            │
                            ▼
┌──────────────────────────────────────────────────────────────┐
│                      ANDROID APPLICATION                     │
│                                                              │
│  onActivityResult():                                         │
│  • Receives Intent with result data                          │
│  • Extracts RESULT_DATA and MESSAGE                          │
│  • Processes returned information                            │
│  • Updates UI / saves data / triggers actions                │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Step 1: Export Unity Project

### 1.1 Configure Unity Build Settings

1. Open Unity Editor
2. Go to **File → Build Settings**
3. Select **Android** platform
4. Click **Switch Platform** (if not already on Android)

### 1.2 Configure Player Settings

Click **Player Settings** button in Build Settings and configure:

#### Company and Product Name
- **Company Name**: `com.yourcompany`
- **Product Name**: `YourAppName`
- **Package Name**: `com.yourcompany.yourapp` *(IMPORTANT: Note this down)*

#### Other Settings
- **Minimum API Level**: 24 or higher
- **Scripting Backend**: IL2CPP (recommended) or Mono
- **Target Architectures**: ARM64 (required), ARMv7 (optional)

### 1.3 Export Project

1. In Build Settings, **check "Export Project"** checkbox
2. **Uncheck "Build App Bundle (Google Play)"** if checked
3. Click **Export** button
4. Choose export location (e.g., `UnityExport`)

### 1.4 What Gets Exported

After export, you'll have an Android Studio project structure:
```
UnityExport/
├── launcher/
├── unityLibrary/     ← Contains your Unity scene project
├── build.gradle
└── gradle.properties
```

---

## Step 2: Build AAR Library

### 2.1 Open in Android Studio

1. Open Android Studio
2. **File → Open** → Select the `UnityExport` folder
3. Wait for Gradle sync to complete

### 2.2 Build the AAR

Open Terminal in Android Studio and run:

```bash
# Navigate to project directory
cd UnityExport

# Build the AAR
./gradlew :unityLibrary:assembleRelease
```

**Output Location:**
```
UnityExport/unityLibrary/build/outputs/aar/unityLibrary-release.aar
```

This is the file you'll integrate into your Android app!

---

## Step 3: Integrate AAR into Android App

### 3.1 Prepare Your Android Project

Create or open your existing Android project in Android Studio.

### 3.2 Add AAR to Project

#### Method: Via libs folder (Recommended)

1. Create a `libs` folder inside `app/`:
   ```
   YourAndroidApp/
   └── app/
       ├── libs/              ← Create this folder
       │   └── unityLibrary-release.aar
       ├── src/
       └── build.gradle
   ```

2. Copy `unityLibrary-release.aar` to `app/libs/`

### 3.3 Update app/build.gradle

Add the AAR dependency:

```gradle
dependencies {
    // Unity Library
    implementation files('libs/unityLibrary-release.aar')

    // Other dependencies...
}
```

### 3.4 Sync Gradle

Click **Sync Now** when prompted or **File → Sync Project with Gradle Files**.

---

## Step 4: Android Implementation (Java)

### Required Changes Summary (Java)

To integrate Unity with two-way communication, you need to:
1. ✅ Update `AndroidManifest.xml` with Unity activity and permissions
2. ✅ Create/Update `MainActivity.java` to launch Unity with Intent
3. ✅ Implement `onActivityResult()` to receive data from Unity
4. ✅ Create layout XML with input field and button

### 4.1 Update AndroidManifest.xml

Add Unity activity and required permissions:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <!-- Required Permissions -->
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.CAMERA" />

    <!-- Required OpenGL ES and Features -->
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
        android:hardwareAccelerated="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.YourApp">

        <!-- Main Activity -->
        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:launchMode="singleTop">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <!-- Unity Activity (REQUIRED) -->
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

### 4.2 Create/Update MainActivity.java

```java
package com.yourcompany.mainapp;

import android.app.Activity;
import android.content.Intent;
import android.os.Bundle;
import android.view.View;
import android.widget.Button;
import android.widget.EditText;
import android.widget.Toast;
import androidx.annotation.Nullable;
import com.unity3d.player.UnityPlayerActivity;

public class MainActivity extends Activity {

    private static final int UNITY_REQUEST_CODE = 1001;
    private EditText mapCodeInput;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mapCodeInput = findViewById(R.id.mapCodeInput);
        Button openNavigationButton = findViewById(R.id.openNavigationButton);

        openNavigationButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                openUnityNavigation();
            }
        });
    }

    /**
     * Launch Unity with MapCode parameter.
     * UnityAndroidBridge will automatically read the MAP_CODE from Intent extras.
     */
    private void openUnityNavigation() {
        String mapCode = mapCodeInput.getText().toString().trim();

        if (mapCode.isEmpty()) {
            Toast.makeText(this, "Please enter a MapCode", Toast.LENGTH_SHORT).show();
            return;
        }

        // Create intent to launch Unity
        Intent intent = new Intent(this, UnityPlayerActivity.class);

        // Pass MapCode to Unity via Intent extras
        // UnityAndroidBridge automatically reads this and applies it
        intent.putExtra("MAP_CODE", mapCode);

        // Launch Unity and expect a result
        startActivityForResult(intent, UNITY_REQUEST_CODE);
    }

    /**
     * Receive result from Unity when it closes.
     * Unity calls ReturnToAndroid() which sends data back here.
     */
    @Override
    protected void onActivityResult(int requestCode, int resultCode, @Nullable Intent data) {
        super.onActivityResult(requestCode, resultCode, data);

        if (requestCode == UNITY_REQUEST_CODE) {
            if (resultCode == RESULT_OK && data != null) {
                // Get data returned from Unity
                String resultData = data.getStringExtra("RESULT_DATA");
                String message = data.getStringExtra("MESSAGE");

                Toast.makeText(this,
                    "Returned from Unity!\nData: " + resultData + "\nMessage: " + message,
                    Toast.LENGTH_LONG).show();

                // Process the returned data
                handleUnityResult(resultData, message);
            } else {
                Toast.makeText(this, "Unity closed without result", Toast.LENGTH_SHORT).show();
            }
        }
    }

    /**
     * Handle data returned from Unity.
     * Implement your business logic here.
     */
    private void handleUnityResult(String resultData, String message) {
        // Process the data as needed
        // Examples:
        // - Update UI
        // - Save to database
        // - Trigger analytics
        // - Navigate to another screen

        System.out.println("Unity Result - Data: " + resultData + ", Message: " + message);

        // Example: Update the input field with returned data
        if (resultData != null) {
            mapCodeInput.setText(resultData);
        }
    }
}
```

### 4.3 Create Layout (res/layout/activity_main.xml)

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
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

## Step 5: Android Implementation (Kotlin)

### Required Changes Summary (Kotlin)

To integrate Unity with two-way communication, you need to:
1. ✅ Update `AndroidManifest.xml` with Unity activity and permissions (same as Java)
2. ✅ Create/Update `MainActivity.kt` to launch Unity with Intent
3. ✅ Implement `onActivityResult()` to receive data from Unity
4. ✅ Create layout XML with input field and button (same as Java)

### 5.1 AndroidManifest.xml

Use the same `AndroidManifest.xml` from Step 4.1 (Java section).

### 5.2 Create/Update MainActivity.kt

```kotlin
package com.yourcompany.mainapp

import android.app.Activity
import android.content.Intent
import android.os.Bundle
import android.widget.Button
import android.widget.EditText
import android.widget.Toast
import com.unity3d.player.UnityPlayerActivity

class MainActivity : Activity() {

    companion object {
        private const val UNITY_REQUEST_CODE = 1001
    }

    private lateinit var mapCodeInput: EditText

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        mapCodeInput = findViewById(R.id.mapCodeInput)
        val openNavigationButton = findViewById<Button>(R.id.openNavigationButton)

        openNavigationButton.setOnClickListener {
            openUnityNavigation()
        }
    }

    /**
     * Launch Unity with MapCode parameter.
     * UnityAndroidBridge will automatically read the MAP_CODE from Intent extras.
     */
    private fun openUnityNavigation() {
        val mapCode = mapCodeInput.text.toString().trim()

        if (mapCode.isEmpty()) {
            Toast.makeText(this, "Please enter a MapCode", Toast.LENGTH_SHORT).show()
            return
        }

        // Create intent to launch Unity
        val intent = Intent(this, UnityPlayerActivity::class.java).apply {
            // Pass MapCode to Unity via Intent extras
            // UnityAndroidBridge automatically reads this and applies it
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
        // Process the data as needed
        // Examples:
        // - Update UI
        // - Save to database
        // - Trigger analytics
        // - Navigate to another screen

        println("Unity Result - Data: $resultData, Message: $message")

        // Example: Update the input field with returned data
        resultData?.let {
            mapCodeInput.setText(it)
        }
    }
}
```

### 5.3 Layout XML

Use the same layout XML from Step 4.3 (Java section).

---

## Advanced Usage Examples

### Example 1: Send Runtime Messages to Unity

#### Java
```java
import com.unity3d.player.UnityPlayer;

public class MainActivity extends Activity {

    /**
     * Send a new MapCode to Unity while it's running
     */
    private void updateMapCodeInRuntime(String newMapCode) {
        // Method 1: Direct method call
        UnityPlayer.UnitySendMessage(
            "UnityAndroidBridge",  // GameObject name
            "SetMapCode",          // Method name
            newMapCode             // Parameter
        );

        // Method 2: Key-value format
        UnityPlayer.UnitySendMessage(
            "UnityAndroidBridge",
            "ReceiveData",
            "mapcode:" + newMapCode
        );

        // Method 3: JSON format
        String jsonData = String.format(
            "{\"messageType\": \"mapcode\", \"payload\": \"%s\"}",
            newMapCode
        );
        UnityPlayer.UnitySendMessage(
            "UnityAndroidBridge",
            "ReceiveJsonData",
            jsonData
        );
    }

    /**
     * Request POI list from Unity
     */
    private void requestPOIList() {
        UnityPlayer.UnitySendMessage(
            "UnityAndroidBridge",
            "GetPOIList",
            ""
        );
    }
}
```

#### Kotlin
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

---

### Example 2: Dynamic MapCode Selection with Spinner

#### Java
```java
import android.widget.Spinner;
import android.widget.ArrayAdapter;

public class MainActivity extends Activity {

    private Spinner mapCodeSpinner;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        setupMapCodeSpinner();
    }

    private void setupMapCodeSpinner() {
        mapCodeSpinner = findViewById(R.id.mapCodeSpinner);

        // Define available map codes
        String[] mapCodes = {
            "MAP_BUILDING_A",
            "MAP_BUILDING_B",
            "MAP_CAMPUS",
            "MAP_FLOOR_1",
            "MAP_FLOOR_2"
        };

        ArrayAdapter<String> adapter = new ArrayAdapter<>(
            this,
            android.R.layout.simple_spinner_item,
            mapCodes
        );
        adapter.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item);
        mapCodeSpinner.setAdapter(adapter);

        Button openButton = findViewById(R.id.openNavigationButton);
        openButton.setOnClickListener(v -> {
            String selectedMapCode = mapCodeSpinner.getSelectedItem().toString();
            launchUnityWithMapCode(selectedMapCode);
        });
    }

    private void launchUnityWithMapCode(String mapCode) {
        Intent intent = new Intent(this, UnityPlayerActivity.class);
        intent.putExtra("MAP_CODE", mapCode);
        startActivityForResult(intent, UNITY_REQUEST_CODE);
    }
}
```

#### Kotlin
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
        val intent = Intent(this, UnityPlayerActivity::class.java)
        intent.putExtra("MAP_CODE", mapCode)
        startActivityForResult(intent, UNITY_REQUEST_CODE)
    }
}
```

---

### Example 3: Handle Multiple Return Data Types

#### Java
```java
import org.json.JSONObject;
import org.json.JSONArray;

public class MainActivity extends Activity {

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);

        if (requestCode == UNITY_REQUEST_CODE && resultCode == RESULT_OK && data != null) {

            // Standard return data
            String resultData = data.getStringExtra("RESULT_DATA");
            String message = data.getStringExtra("MESSAGE");

            // Check if we received JSON data
            if (resultData != null && resultData.startsWith("{")) {
                handleJsonResult(resultData);
            } else {
                handleStandardResult(resultData, message);
            }
        }
    }

    private void handleJsonResult(String jsonData) {
        try {
            JSONObject jsonObject = new JSONObject(jsonData);
            JSONArray pois = jsonObject.getJSONArray("pois");

            System.out.println("Received " + pois.length() + " POIs from Unity");

            for (int i = 0; i < pois.length(); i++) {
                JSONObject poi = pois.getJSONObject(i);
                String name = poi.getString("name");
                System.out.println("POI: " + name);
            }

        } catch (Exception e) {
            System.out.println("Error parsing JSON: " + e.getMessage());
        }
    }

    private void handleStandardResult(String resultData, String message) {
        System.out.println("Standard result - Data: " + resultData + ", Message: " + message);

        // Update UI or perform actions
        if (resultData != null) {
            // Save to preferences, update database, etc.
        }
    }
}
```

#### Kotlin
```kotlin
import org.json.JSONObject

class MainActivity : Activity() {

    override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
        super.onActivityResult(requestCode, resultCode, data)

        if (requestCode == UNITY_REQUEST_CODE && resultCode == RESULT_OK && data != null) {

            // Standard return data
            val resultData = data.getStringExtra("RESULT_DATA")
            val message = data.getStringExtra("MESSAGE")

            // Check if we received JSON data
            if (resultData?.startsWith("{") == true) {
                handleJsonResult(resultData)
            } else {
                handleStandardResult(resultData, message)
            }
        }
    }

    private fun handleJsonResult(jsonData: String) {
        try {
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

        // Update UI or perform actions
        resultData?.let {
            // Save to preferences, update database, etc.
        }
    }
}
```

---

## Quick Start Checklist

### Unity Side
- [ ] Ensure UnityAndroidBridge.cs is in your scene
- [ ] Configure Build Settings for Android
- [ ] Set Package Name in Player Settings
- [ ] Enable "Export Project" in Build Settings
- [ ] Export Unity project
- [ ] Build AAR using Gradle

### Android Side
- [ ] Copy AAR to `app/libs/` folder
- [ ] Update `build.gradle` with AAR dependency
- [ ] Add Unity activity to `AndroidManifest.xml`
- [ ] Add required permissions (INTERNET, CAMERA)
- [ ] Add required features (OpenGL ES, touchscreen)
- [ ] Create MainActivity (Java or Kotlin)
- [ ] Implement Intent launch with `putExtra("MAP_CODE", ...)`
- [ ] Implement `onActivityResult()` to receive data
- [ ] Create layout XML with MapCode input
- [ ] Build and test

---

## Summary: What Changes Are Required on Android Side

### 1. AndroidManifest.xml
**Required:**
- Unity activity declaration (`com.unity3d.player.UnityPlayerGameActivity`)
- Permissions: `INTERNET`, `CAMERA`
- Features: OpenGL ES 3.0, touchscreen, camera

### 2. build.gradle
**Required:**
- Add Unity AAR dependency: `implementation files('libs/unityLibrary-release.aar')`

### 3. MainActivity (Java/Kotlin)
**Required:**
- Create Intent to launch `UnityPlayerActivity`
- Pass MapCode via `intent.putExtra("MAP_CODE", mapCode)`
- Use `startActivityForResult()` to expect data back
- Override `onActivityResult()` to receive returned data
- Extract `RESULT_DATA` and `MESSAGE` from returned Intent

### 4. Layout XML
**Required:**
- Input field for MapCode (`EditText`)
- Button to launch Unity
- (Optional) TextView to display results

### 5. Unity Side
**No changes needed!**
- UnityAndroidBridge handles everything automatically
- Reads Intent extras on launch
- Applies MapCode to MapLocalizationManager
- Returns data on back button press

---

## Key Integration Points

| Direction | Android Code | Unity Code (Automatic) | Data Keys |
|-----------|-------------|----------------------|-----------|
| Android → Unity | `intent.putExtra("MAP_CODE", mapCode)` | `ReadMapCodeFromIntent()` | `MAP_CODE` |
| Unity → Android | `onActivityResult(...)` | `ReturnToAndroid(data, msg)` | `RESULT_DATA`, `MESSAGE` |
| Runtime Update | `UnityPlayer.UnitySendMessage(...)` | `SetMapCode()`, `ReceiveData()` | Various |

---

Generated for MultiSet SDK Unity-Android Integration
