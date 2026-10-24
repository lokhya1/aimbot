settings.gradle
build.gradle
app/build.gradle
app/proguard-rules.pro
app/src/main/AndroidManifest.xml
app/src/main/java/com/example/gamingstability/MainActivity.java
app/src/main/java/com/example/gamingstability/OverlayService.java
app/src/main/res/values/styles.xml

settings.gradle
----------------
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}

rootProject.name = "GamingStabilityOverlay"
include(":app")

build.gradle
------------
plugins {
    id 'com.android.application' version '8.6.1' apply false
}

app/build.gradle
----------------
plugins {
    id 'com.android.application'
}

android {
    namespace 'com.example.gamingstability'
    compileSdk 35

    defaultConfig {
        applicationId "com.example.gamingstability"
        minSdk 26
        targetSdk 35
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }

    compileOptions {
        sourceCompatibility JavaVersion.VERSION_17
        targetCompatibility JavaVersion.VERSION_17
    }
}

dependencies {
    implementation 'androidx.appcompat:appcompat:1.7.0'
}

app/proguard-rules.pro
----------------------
# Keep project defaults

app/src/main/AndroidManifest.xml
-------------------------------
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

    <application
        android:allowBackup="false"
        android:label="Gaming Stability Overlay"
        android:theme="@style/AppTheme">

        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <service
            android:name=".OverlayService"
            android:exported="false" />
    </application>

</manifest>

app/src/main/java/com/example/gamingstability/MainActivity.java
------------------------------------------------------------
package com.example.gamingstability;

import android.app.Activity;
import android.content.Intent;
import android.net.Uri;
import android.os.Bundle;
import android.provider.Settings;
import android.view.Gravity;
import android.view.ViewGroup;
import android.widget.Button;
import android.widget.LinearLayout;
import android.widget.TextView;

public class MainActivity extends Activity {
    private static final int OVERLAY_REQUEST = 101;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        LinearLayout root = new LinearLayout(this);
        root.setOrientation(LinearLayout.VERTICAL);
        root.setPadding(32, 36, 32, 32);
        root.setGravity(Gravity.CENTER_HORIZONTAL);
        root.setBackgroundColor(0xFFF5F7FA);

        TextView title = new TextView(this);
        title.setText("Gaming Stability Overlay");
        title.setTextSize(24);
        title.setTextColor(0xFF152238);
        title.setPadding(0, 0, 0, 18);
        root.addView(title);

        TextView info = new TextView(this);
        info.setText("A simple floating helper panel for gaming. It shows a timer and safe reminders; it does not change Free Fire, measure the game's true FPS, or automate aiming.\n\n1. Allow 'Display over other apps'.\n2. Tap Start Overlay.\n3. Open your game. Move the panel by dragging it.");
        info.setTextSize(16);
        info.setTextColor(0xFF263238);
        info.setPadding(0, 0, 0, 24);
        root.addView(info);

        Button permission = new Button(this);
        permission.setText("Allow Display Over Other Apps");
        permission.setOnClickListener(v -> {
            Intent intent = new Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION,
                    Uri.parse("package:" + getPackageName()));
            startActivityForResult(intent, OVERLAY_REQUEST);
        });
        root.addView(permission, new LinearLayout.LayoutParams(
                ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT));

        Button start = new Button(this);
        start.setText("Start Overlay");
        start.setOnClickListener(v -> {
            if (Settings.canDrawOverlays(this)) {
                startService(new Intent(this, OverlayService.class));
            } else {
                startActivity(new Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION,
                        Uri.parse("package:" + getPackageName())));
            }
        });
        root.addView(start, new LinearLayout.LayoutParams(
                ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT));

        Button stop = new Button(this);
        stop.setText("Stop Overlay");
        stop.setOnClickListener(v -> stopService(new Intent(this, OverlayService.class)));
        root.addView(stop, new LinearLayout.LayoutParams(
                ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT));

        setContentView(root);
    }
}

app/src/main/java/com/example/gamingstability/OverlayService.java
--------------------------------------------------------------
package com.example.gamingstability;

import android.app.Notification;
import android.app.NotificationChannel;
import android.app.NotificationManager;
import android.app.Service;
import android.content.Intent;
import android.graphics.Color;
import android.graphics.PixelFormat;
import android.os.Build;
import android.os.Handler;
import android.os.IBinder;
import android.view.Gravity;
import android.view.MotionEvent;
import android.view.View;
import android.view.WindowManager;
import android.widget.LinearLayout;
import android.widget.TextView;

import java.util.Locale;

public class OverlayService extends Service {
    private WindowManager windowManager;
    private View overlay;
    private TextView timerText;
    private final Handler handler = new Handler();
    private long startedAt;

    private final Runnable ticker = new Runnable() {
        @Override public void run() {
            if (timerText != null) {
                long seconds = (System.currentTimeMillis() - startedAt) / 1000L;
                timerText.setText(String.format(Locale.getDefault(),
                        "GAME HELPER\nSession %02d:%02d\nSmooth graphics • Stay cool\nDrag to move • Tap ✕ to close",
                        seconds / 60, seconds % 60));
                handler.postDelayed(this, 1000);
            }
        }
    };

    @Override public void onCreate() {
        super.onCreate();

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            NotificationChannel channel = new NotificationChannel(
                    "gaming_overlay",
                    "Gaming overlay",
                    NotificationManager.IMPORTANCE_LOW
            );
            NotificationManager nm = getSystemService(NotificationManager.class);
            if (nm != null) nm.createNotificationChannel(channel);

            Notification notification = new Notification.Builder(this, "gaming_overlay")
                    .setContentTitle("Gaming Stability Overlay")
                    .setContentText("Floating helper is running")
                    .setSmallIcon(android.R.drawable.ic_menu_info_details)
                    .build();

            startForeground(11, notification);
        }

        windowManager = (WindowManager) getSystemService(WINDOW_SERVICE);

        LinearLayout panel = new LinearLayout(this);
        panel.setOrientation(LinearLayout.HORIZONTAL);
        panel.setPadding(18, 12, 10, 12);
        panel.setBackgroundColor(0xE6222633);

        timerText = new TextView(this);
        timerText.setTextColor(Color.WHITE);
        timerText.setTextSize(12);
        timerText.setText("GAME HELPER\nStarting...");
        panel.addView(timerText);

        TextView close = new TextView(this);
        close.setText("  ✕ ");
        close.setTextColor(Color.WHITE);
        close.setTextSize(16);
        close.setPadding(12, 0, 6, 0);
        panel.addView(close);
        close.setOnClickListener(v -> stopSelf());

        int type = Build.VERSION.SDK_INT >= Build.VERSION_CODES.O
                ? WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY
                : WindowManager.LayoutParams.TYPE_PHONE;

        WindowManager.LayoutParams params = new WindowManager.LayoutParams(
                WindowManager.LayoutParams.WRAP_CONTENT,
                WindowManager.LayoutParams.WRAP_CONTENT,
                type,
                WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                        | WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN,
                PixelFormat.TRANSLUCENT
        );
        params.gravity = Gravity.TOP | Gravity.START;
        params.x = 24;
        params.y = 160;

        panel.setOnTouchListener(new View.OnTouchListener() {
            private int startX, startY;
            private float touchX, touchY;

            @Override
            public boolean onTouch(View v, MotionEvent event) {
                switch (event.getAction()) {
                    case MotionEvent.ACTION_DOWN:
                        startX = params.x;
                        startY = params.y;
                        touchX = event.getRawX();
                        touchY = event.getRawY();
                        return true;

                    case MotionEvent.ACTION_MOVE:
                        params.x = startX + (int) (event.getRawX() - touchX);
                        params.y = startY + (int) (event.getRawY() - touchY);
                        try {
                            windowManager.updateViewLayout(panel, params);
                        } catch (Exception ignored) {
                        }
                        return true;

                    default:
                        return false;
                }
            }
        });

        overlay = panel;

        try {
            windowManager.addView(overlay, params);
        } catch (Exception e) {
            stopSelf();
            return;
        }

        startedAt = System.currentTimeMillis();
        handler.post(ticker);
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        return START_NOT_STICKY;
    }

    @Override
    public void onDestroy() {
        handler.removeCallbacks(ticker);
        if (windowManager != null && overlay != null) {
            try {
                windowManager.removeView(overlay);
            } catch (Exception ignored) {
            }
        }
        overlay = null;
        super.onDestroy();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}

app/src/main/res/values/styles.xml
---------------------------------
<resources>
    <style name="AppTheme" parent="@android:style/Theme.Material.Light.NoActionBar">
        <item name="android:fontFamily">sans</item>
        <item name="android:colorAccent">#1565C0</item>
        <item name="android:windowLightStatusBar">true</item>
        <item name="android:windowActionModeOverlay">true</item>
    </style>
</resources>
