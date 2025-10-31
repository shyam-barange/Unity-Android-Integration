
Documentation for Android Unity Integration

----------------------------------------------------------------------------------------------------------------------

Step 1 : Switch the Platform to Android in Unity Editor, Export Unity Project as Android Project from build settings, check 'Export Project' option.

----------------------------------------------------------------------------------------------------------------------
Step 2 : Open the exported project : 
         it contains following modules : 
            -launcher
            -unityLibrary
            -shared

----------------------------------------------------------------------------------------------------------------------
Step 3 : Now Create new Android project or open existing Android project

----------------------------------------------------------------------------------------------------------------------

Step 4 : Add the unityLibrary and shared module to the root directory of android project 

----------------------------------------------------------------------------------------------------------------------

Step 5 : update the settings.gradle file in the root directory of android project 

settings.gradle(Project Settings)----------

include ':unityLibrary'
project(':unityLibrary').projectDir=new File('...Path..\\AndroidStudioProjects\\AndroidProject\\unityLibrary')
include 'unityLibrary:xrmanifest.androidlib'

-------------
Step 6 : Update the build.gradle file in the app module

build.gradle(Module :app)----------

implementation project(':unityLibrary')
implementation fileTree(dir: project(':unityLibrary').getProjectDir().toString() + ('\\libs'), include: ['*.jar'])

----------------------------------------------------------------------------------------------------------------------

Step 7 : Update the gradle.properties file in the root directory of android project

gradle.properties(Project Properties)-------------

unityStreamingAssets=.unity3d, google-services-desktop.json, google-services.json, GoogleService-Info.plist

----------------------------------------------------------------------------------------------------------------------

Step 8 : Update the build.gradle file in the unityLibrary module, make sure the namespace is 'com.unity3d.player'

build.gradle(module:unityLibrary)------------

namespace 'com.unity3d.player'

----------------------------------------------------------------------------------------------------------------------

Step 9 : Update the strings.xml file in the app module (optional)

strings.xml(app)---------

<string name="game_view_content_description">Game view</string>

----------------------------------------------------------------------------------------------------------------------

Step 10 : Update the android.manifest file in the unityLibrary module

android.manifest(unityLibrary)-------------

<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

    <uses-feature android:glEsVersion="0x00030000" />
    <uses-feature
        android:name="android.hardware.location.gps"
        android:required="false" />
    <uses-feature
        android:name="android.hardware.location"
        android:required="false" />
    <uses-feature
        android:name="android.hardware.touchscreen"
        android:required="false" />
    <uses-feature
        android:name="android.hardware.touchscreen.multitouch"
        android:required="false" />
    <uses-feature
        android:name="android.hardware.touchscreen.multitouch.distinct"
        android:required="false" />

    <application
        android:appCategory="game"
        android:enableOnBackInvokedCallback="true"
        android:extractNativeLibs="true">
        <meta-data
            android:name="unity.splash-mode"
            android:value="0" />
        <meta-data
            android:name="unity.splash-enable"
            android:value="True" />
        <meta-data
            android:name="unity.launch-fullscreen"
            android:value="True" />
        <meta-data
            android:name="unity.render-outside-safearea"
            android:value="True" />
        <meta-data
            android:name="notch.config"
            android:value="portrait|landscape" />
        <meta-data
            android:name="unity.auto-report-fully-drawn"
            android:value="true" />
        <meta-data
            android:name="unity.strip-engine-code"
            android:value="true" />
        <meta-data
            android:name="unity.auto-set-game-state"
            android:value="true" />

        <activity
            android:name="com.unity3d.player.UnityPlayerGameActivity"
            android:configChanges="mcc|mnc|locale|touchscreen|keyboard|keyboardHidden|navigation|orientation|screenLayout|uiMode|screenSize|smallestScreenSize|fontScale|layoutDirection|density"
            android:enabled="true"
            android:exported="true"
            android:hardwareAccelerated="false"
            android:launchMode="singleTask"
            android:resizeableActivity="true"
            android:screenOrientation="fullUser"

            android:process=":unityplayer"

            android:theme="@style/BaseUnityGameActivityTheme">
<!--            <intent-filter>-->
<!--                <category android:name="android.intent.category.LAUNCHER" />-->
<!--                <action android:name="android.intent.action.MAIN" />-->
<!--            </intent-filter>-->


            <meta-data
                android:name="unityplayer.UnityActivity"
                android:value="true" />
            <meta-data
                android:name="android.app.lib_name"
                android:value="game" />
            <meta-data
                android:name="WindowManagerPreference:FreeformWindowSize"
                android:value="@string/FreeformWindowSize_maximize" />
            <meta-data
                android:name="WindowManagerPreference:FreeformWindowOrientation"
                android:value="@string/FreeformWindowOrientation_landscape" />
            <meta-data
                android:name="notch_support"
                android:value="true" />
        </activity>

        <meta-data
            android:name="unityplayer.SkipPermissionsDialog"
            android:value="true" />
        <meta-data
            android:name="com.google.ar.core"
            android:value="required" />
    </application>
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-feature
        android:name="android.hardware.camera.ar"
        android:required="true" />
</manifest>

----------------------------------------------------------------------------------------------------------------------
Main change in the activity tag of unityLibrary module is : 

<activity
            android:name="com.unity3d.player.UnityPlayerGameActivity"
            android:configChanges="mcc|mnc|locale|touchscreen|keyboard|keyboardHidden|navigation|orientation|screenLayout|uiMode|screenSize|smallestScreenSize|fontScale|layoutDirection|density"
            android:enabled="true"
            android:exported="true"
            android:hardwareAccelerated="false"
            android:launchMode="singleTask"
            android:resizeableActivity="true"
            android:screenOrientation="fullUser"

----------------------------------------------------------------------------------------------------------------------
            android:process=":unityplayer"
----------------------------------------------------------------------------------------------------------------------


            android:theme="@style/BaseUnityGameActivityTheme">
<!--            <intent-filter>-->
<!--                <category android:name="android.intent.category.LAUNCHER" />-->
<!--                <action android:name="android.intent.action.MAIN" />-->
<!--            </intent-filter>-->


            <meta-data
                android:name="unityplayer.UnityActivity"
                android:value="true" />
            <meta-data
                android:name="android.app.lib_name"
                android:value="game" />
            <meta-data
                android:name="WindowManagerPreference:FreeformWindowSize"
                android:value="@string/FreeformWindowSize_maximize" />
            <meta-data
                android:name="WindowManagerPreference:FreeformWindowOrientation"
                android:value="@string/FreeformWindowOrientation_landscape" />
            <meta-data
                android:name="notch_support"
                android:value="true" />
        </activity>
----------------------------------------------------------------------------------------------------------------------

Step 11 : Update the build.gradle file in the unityLibrary module   

Update following lines in build.gradle file : 

    implementation fileTree(dir: 'libs', include: ['*.jar'])

    implementation files('libs/arcore_client.aar')
    implementation files('libs/ARPresto.aar')
    implementation files('libs/UnityARCore.aar')
    implementation files('libs/unityandroidpermissions.aar')
    implementation project('xrmanifest.androidlib')

// some how existing code throws error, so comment / remove it
//    implementation(name: 'arcore_client', ext:'aar')
//    implementation(name: 'UnityARCore', ext:'aar')
//    implementation(name: 'ARPresto', ext:'aar')
//    implementation(name: 'unityandroidpermissions', ext:'aar')
//    implementation project(':unityLibrary:xrmanifest.androidlib')

----------------------------------------------------------------------------------------------------------------------

Step 12 : Opening Unity Scene from Android activity : 

    public void OpenUnity(View view){
        Intent unityIntent = new Intent(this, UnityPlayerGameActivity.class);
        startActivity(unityIntent);
    }

----------------------------------------------------------------------------------------------------------------------

Step 13 : Opening Unity Scene from Android activity and pass data : 

    // Open unity scene and pass data..

    int LAUNCH_UNITY = 101;

    public void OpenUnity(View view) {
        Intent unityIntent = new Intent(this, UnityPlayerGameActivity.class);
        unityIntent.putExtra("result", "MultisetSDK");
        startActivityForResult(unityIntent, LAUNCH_UNITY);
    }
----------------------------------------------------------------------------------------------------------------------


Step 14 : Receive data from unity app;

    //Receive data from unity app;
    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);

        try {
            if (requestCode == LAUNCH_UNITY && resultCode == RESULT_OK) {
                String requiredValue = data.getStringExtra("result");
                received_data.setText(requiredValue);
            }
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }


----------------------------------------------------------------------------------------------------------------------

Step 15 : Send data from android to unity;


  //TODO: inside UnityPlayerGameActivity :
    public String GetDataFromAndroid() {
        return getIntent().getStringExtra("result");
    }

    public void SetDataFromUnity(String cmdLine) {
        Log.wtf("unity", cmdLine);
        Intent returnIntent = new Intent();
        returnIntent.putExtra("result", cmdLine);
        setResult(Activity.RESULT_OK, returnIntent);
        finish();
    }
----------------------------------------------------------------------------------------------------------------------


