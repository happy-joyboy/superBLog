+++
date = '2025-07-30T00:55:36+03:00'
draft = false
title = 'Android Services - Complete Development & Security Guide'
tags = ['Summary', 'Andorid', 'services', '']
+++

## Summary

> Android Services are application components designed to perform long-running operations in the background without providing a user interface. They enable apps to continue executing tasks even when users switch to other applications or when the app is closed.

> [!Who can use Services?]
> 
> 1. **Activities**: Can start and bind to services for background operations
> 2. **Other Services**: Services can communicate with each other
> 3. **Broadcast Receivers**: Can trigger service operations based on system events
> 4. **Content Providers**: Can use services for data synchronization
> 5. **System Components**: Android system can restart services based on configuration
> 6. **External Apps**: Through proper permissions and exported services

## What Services Look Like

### Key Components of a Service

A Service consists of several lifecycle methods and configuration options:

```java
public class ExampleService extends Service {
    private MediaPlayer player;
    private final IBinder binder = new LocalBinder();
    
    // Lifecycle callbacks
    @Override
    public void onCreate() {
        super.onCreate();
        // One-time setup
    }
    
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        // Handle start requests
        return START_STICKY; // or START_NOT_STICKY, START_REDELIVER_INTENT
    }
    
    @Override
    public IBinder onBind(Intent intent) {
        return binder; // Return IBinder for bound services
    }
    
    @Override
    public boolean onUnbind(Intent intent) {
        return super.onUnbind(intent);
    }
    
    @Override
    public void onRebind(Intent intent) {
        super.onRebind(intent);
    }
    
    @Override
    public void onDestroy() {
        super.onDestroy();
        // Cleanup resources
    }
}
```

> [!Warning] Threading Considerations Services run on the **main thread** by default. For intensive operations, create separate threads to avoid ANR (Application Not Responding) errors.

## Job Service

A very common service you might see exposed is an [Android Job Scheduler](https://developer.android.com/reference/android/app/job/JobScheduler) service. However due to the `android.permission.BIND_JOB_SERVICE` permission this service cannot be directly interacted with and can usually be ignored when hunting for bugs.

### AndroidManifest.xml

```xml
<service android:name=".MyJobService" 
 android:permission="android.permission.BIND_JOB_SERVICE"
 android:exported="true"></service>
```

![[jobService.png]]


## Types of Services

### 1. Started Services (Unbound)

_Services that run independently after being started_
when bound method returns nothing or throws an exception 
- **Lifecycle**: Started with `startService()`, runs indefinitely until stopped
- **Use Cases**: Music playback, file downloads, data synchronization etc...
- **Stopping**: Must be explicitly stopped with `stopSelf()` or `stopService()`

ex:
```java
public class MusicService extends Service {
    private MediaPlayer player;
    
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        player = MediaPlayer.create(this, Settings.System.DEFAULT_RINGTONE_URI);
        player.setLooping(true);
        player.start();
        
        return START_STICKY; // Restart if killed by system
    }
    
    @Override
    public void onDestroy() {
        super.onDestroy();
        if (player != null) {
            player.stop();
            player.release();
        }
    }
    
    @Override
    public IBinder onBind(Intent intent) {
        return null; // Not providing binding
    }
}
```

**Starting from Activity:**

```java
// Start service
Intent serviceIntent = new Intent(this, MusicService.class);
startService(serviceIntent);

// Stop service  
stopService(new Intent(this, MusicService.class));
```

#### Return Flags for onStartCommand()

|Flag|Behavior|
|---|---|
|`START_NOT_STICKY`|Don't recreate service if killed|
|`START_STICKY`|Recreate service but don't redeliver intent|
|`START_REDELIVER_INTENT`|Recreate service and redeliver last intent|

### 2. Bound Services

_Services that provide client-server interface for interaction_

They get started to execute something in the background, and so called [bound Services](https://developer.android.com/develop/background-work/services/bound-services) where an app can establish a connection and continuously exchange data with the other app. SO using `onstartcommand` is like a one way communication but now we can send & receive data

- **Lifecycle**: Lives as long as components are bound to it
- **Use Cases**: IPC, data sharing, remote method calls
- **Communication**: Through `IBinder` interface

#### Binding Methods

>[!error] When talking to bound services with `bindService()` you will need to send an intent but also you will need to setup a `ServiceConnection` and pass it in `bindService(intent,ServiceConnection)`

```java
private ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName className, IBinder service) {
        }
        
        @Override
        public void onServiceDisconnected(ComponentName arg0) {
        }
    };
```

##### A. Extending Binder Class (Local Services)

>[!note] we as attackers can't bind to this service since only components inside app are allowed to bind to it and no one else 

**Best for same-process communication**

```java
public class LocalService extends Service {
    private final IBinder binder = new LocalBinder();
    
    public class LocalBinder extends Binder {
        LocalService getService() {
            return LocalService.this;
        }
    }
    
    @Override
    public IBinder onBind(Intent intent) {
        return binder;
    }
}
```

**Client Activity:**

```java
public class BindingActivity extends Activity {
    LocalService mService;
    boolean mBound = false;
    
    private ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName className, IBinder service) {
            LocalService.LocalBinder binder = (LocalService.LocalBinder) service;
            mService = binder.getService();
            mBound = true;
        }
        
        @Override
        public void onServiceDisconnected(ComponentName arg0) {
            mBound = false;
        }
    };
    
    @Override
    protected void onStart() {
        super.onStart();
        Intent intent = new Intent(this, LocalService.class);
        bindService(intent, connection, Context.BIND_AUTO_CREATE);
    }
    
    @Override
    protected void onStop() {
        super.onStop();
        if (mBound) {
            unbindService(connection);
            mBound = false;
        }
    }
}
```


##### B. Using Messenger (Cross-Process)

**Best for simple IPC without thread safety concerns**

>[!info] A messenger service can easily be recognised by looking at the `onBind()` method that returns a IBinder object created from the Messenger class.

>[!note] The inline class extending [`Handler`](https://developer.android.com/reference/android/os/Handler) is contains a `handleMessage()` method that implements the actual service logic. The attacker can control the `Message` coming in.

```java
public class MessengerService extends Service {
    static final int MSG_SAY_HELLO = 1;
    
    static class IncomingHandler extends Handler {
        private Context applicationContext;
        
        IncomingHandler(Context context) {
            applicationContext = context.getApplicationContext();
        }
        
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_SAY_HELLO:
                    Toast.makeText(applicationContext, "Hello!", Toast.LENGTH_SHORT).show();
                    break;
                default:
                    super.handleMessage(msg);
            }
        }
    }
    
    Messenger mMessenger;
    
    @Override
    public IBinder onBind(Intent intent) {
        mMessenger = new Messenger(new IncomingHandler(this));
        return mMessenger.getBinder();
    }
}
```

**Client using Messenger:**

```java
public class ActivityMessenger extends Activity {
    Messenger mService = null;
    boolean bound;
    
    private ServiceConnection mConnection = new ServiceConnection() {
        public void onServiceConnected(ComponentName className, IBinder service) {
            mService = new Messenger(service);
            bound = true;
        }
        
        public void onServiceDisconnected(ComponentName className) {
            mService = null;
            bound = false;
        }
    };
    
    public void sayHello(View v) {
        if (!bound) return;
        Message msg = Message.obtain(null, MessengerService.MSG_SAY_HELLO, 0, 0);
        try {
            mService.send(msg);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }
}
```

_using looper_ 
```java
```java
public class MyMessageService extends Service {
    public static final int MSG_SUCCESS = 42;
    final Messenger messenger = new Messenger(new IncomingHandler(Looper.getMainLooper()));

    @Override // android.app.Service
    public IBinder onBind(Intent intent) {
        return this.messenger.getBinder();
    }

    class IncomingHandler extends Handler {

        IncomingHandler(Looper looper) {
            super(looper);
        }

        @Override // android.os.Handler
        public void handleMessage(Message message) {
            if (message.what == 42) {
                // ...
            } else {
                super.handleMessage(message);
            }
        }
    }
}
```

A Looper is just a message pump that Runs an infinite loop on a thread to Pull messages from a MessageQueue then Dispatches/send them to the appropriate Handler and Keeps the thread alive to process messages

##### C. Using AIDL (Android Interface Definition Language)

**Best for complex multi-threaded IPC**

**Step 1: Create AIDL file** (`IRemoteService.aidl`):

To add aidl files you need to set it up in the the gradle build file and search for `buildFeatures` and if you cannot find this block, create it and set aidl to true

```gradle
buildFeatures {  
    aidl true  
}
```

```aidl
// IRemoteService.aidl
package com.example.android;

interface IRemoteService {
    int getPid();
    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, String aString);
}
```

**Step 2: Implement Service:**

```java
public class RemoteService extends Service {
    
    @Override
    public IBinder onBind(Intent intent) {
        return binder;
    }
    
    private final IRemoteService.Stub binder = new IRemoteService.Stub() {
        public int getPid() {
            return Process.myPid();
        }
        
        public void basicTypes(int anInt, long aLong, boolean aBoolean, 
                             float aFloat, double aDouble, String aString) {
            // Implementation
        }
    };
}
```

**Step 3: Client Implementation:**

```java
IRemoteService iRemoteService;

private ServiceConnection mConnection = new ServiceConnection() {
    public void onServiceConnected(ComponentName className, IBinder service) {
        iRemoteService = IRemoteService.Stub.asInterface(service);
    }
    
    public void onServiceDisconnected(ComponentName className) {
        iRemoteService = null;
    }
};
```

##### 2. if the aidl already defined but in different app 

1- if you will use this file to talk to service of a different app you need to set the name exactly as in the service , then 
1. right click on aidl folder and create a new package with the name in `DESCRIPTOR`
2. click on the aidl file you created then choose refactor, move it in the folder you have ejus created 
3. in the file you are in change the package name from your app to the one in `DESCRIPTOR`
4. rebuild the app

2- using class Loading By loading the class directly from the target app, we can just invoke the functions we need and do not have to bother about method order or package names.

```java
ServiceConnection serviceConnection4 = new ServiceConnection() {  
    
    @Override  
    public void onServiceConnected(ComponentName name, IBinder service) {  
        // Load the class dynamically  
        ClassLoader classLoader = null;  
        try {  
            classLoader = services.this.createPackageContext("package.name",  
                    Context.CONTEXT_INCLUDE_CODE | Context.CONTEXT_IGNORE_SECURITY).getClassLoader();  
            Class<?> iRemoteServiceClass;  
            // Load the AIDL interface class  
            iRemoteServiceClass = classLoader.loadClass("class.you.want.to.use");  
            Class<?> stubClass = null;  
            for (Class<?> innerClass : iRemoteServiceClass.getDeclaredClasses()) {  
                if (innerClass.getSimpleName().equals("Stub")) {  
                    stubClass = innerClass;  
                    break;  
                }  
            }  
            // Get the asInterface method  
            Method asInterfaceMethod = stubClass.getDeclaredMethod("asInterface", IBinder.class);  
  
 // Invoke the asInterface method to get the instance of IRemoteService
        Object iRemoteService = asInterfaceMethod.invoke(null, service);

        //Example to create and call method from class
        Method openFlagMethod = iRemoteServiceClass.getDeclaredMethod("openFlag");
        boolean initResult = (boolean) openFlagMethod.invoke(iRemoteService);
  
  
        } catch (Exception e) {  
            Log.e("Flag29Service", "Error creating package context", e);  
        }  
    }  
  
    @Override  
    public void onServiceDisconnected(ComponentName name) {  
    }  
};  
  
bindService(send,serviceConnection4, Context.BIND_AUTO_CREATE);
```

##### Reversing
When you see `.Stub()` related code inside a Service implementation, we probably have an AIDL service. To reverse engineer the original .aidl code, we can look into the generated service interface code.

1. Look for the `DESCRIPTOR` variable, as it contains the original package path and .aidl filename
2. The AIDL methods can be derived from the interface methods with the `throws RemoteException`
3. The original method order is shown by the `TRANSACTION_` integers

### 3. Foreground Services

_Services that perform user-noticeable operations with persistent notifications_

```java
public class ForegroundService extends Service {
    private static final int NOTIFICATION_ID = 1;
    private static final String CHANNEL_ID = "ForegroundServiceChannel";
    
    @Override
    public void onCreate() {
        super.onCreate();
        createNotificationChannel();
    }
    
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        createNotificationChannel();
        
        Intent notificationIntent = new Intent(this, MainActivity.class);
        PendingIntent pendingIntent = PendingIntent.getActivity(this, 0, 
            notificationIntent, PendingIntent.FLAG_IMMUTABLE);
        
        Notification notification = new NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle("Foreground Service")
            .setContentText("Service is running...")
            .setSmallIcon(R.drawable.ic_notification)
            .setContentIntent(pendingIntent)
            .build();
        
        startForeground(NOTIFICATION_ID, notification);
        
        return START_NOT_STICKY;
    }
    
    private void createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            NotificationChannel serviceChannel = new NotificationChannel(
                CHANNEL_ID,
                "Foreground Service Channel",
                NotificationManager.IMPORTANCE_DEFAULT
            );
            
            NotificationManager manager = getSystemService(NotificationManager.class);
            manager.createNotificationChannel(serviceChannel);
        }
    }
    
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```

**Starting Foreground Service:**

```java
// For API 26+
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    startForegroundService(new Intent(this, ForegroundService.class));
} else {
    startService(new Intent(this, ForegroundService.class));
}
```

## Service Security Vulnerabilities & Attacks

### 1. Intent Redirection Attacks

> [!danger] Critical Vulnerability A publicly accessible service might receive an `Intent` object as an extra and then use unsafe methods like `startActivity()`, `sendBroadcast()`, or `startService()` on that embedded `Intent`. This is similar to an Open Redirect in web security.

**Attack Scenario:** An attacker can craft a malicious `Intent` containing a target `Intent` for a normally non-exported component within the vulnerable app. The proxy service then unknowingly forwards this malicious `Intent`, bypassing Android's access restrictions.

**Impact:**

- Theft of authentication details (user session)
- Forging of content within the app
- Arbitrary code execution by rewriting files and substituting native libraries
- Access to Content Providers with `android:grantUriPermissions="true"`

**Vulnerable Code Example:**

```java
public class VulnerableProxyService extends Service {
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        // DANGEROUS: Directly forwarding received intent
        Intent embeddedIntent = intent.getParcelableExtra("target_intent");
        if (embeddedIntent != null) {
            startActivity(embeddedIntent); // Bypasses access controls!
        }
        return START_NOT_STICKY;
    }
    
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```

**Secure Implementation:**

```java
public class SecureProxyService extends Service {
    private static final String[] ALLOWED_ACTIONS = {
        Intent.ACTION_VIEW,
        Intent.ACTION_SEND
    };
    
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        // Extract only necessary and safe data
        String action = intent.getStringExtra("safe_action");
        String data = intent.getStringExtra("safe_data");
        
        if (!isAllowedAction(action) || !isSafeData(data)) {
            Log.w(TAG, "Unsafe intent parameters detected");
            return START_NOT_STICKY;
        }
        
        // Create new intent with validated data only
        Intent safeIntent = new Intent(action);
        safeIntent.setData(Uri.parse(data));
        
        // Remove unsafe flags
        safeIntent.setFlags(0);
        
        // Add only necessary safe flags
        safeIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        
        startActivity(safeIntent);
        return START_NOT_STICKY;
    }
    
    private boolean isAllowedAction(String action) {
        return action != null && Arrays.asList(ALLOWED_ACTIONS).contains(action);
    }
    
    private boolean isSafeData(String data) {
        if (data == null) return false;
        try {
            Uri uri = Uri.parse(data);
            String scheme = uri.getScheme();
            return "https".equals(scheme) || "http".equals(scheme);
        } catch (Exception e) {
            return false;
        }
    }
    
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```

### 2. Improper Platform Usage and Permission-Based Access Control

> [!danger] Permission Bypass An exported service performing sensitive tasks might lack proper `android:permission` or manual permission checks, allowing malicious apps to abuse the service's privileges.

**Vulnerable Manifest:**

```xml
<!-- DANGEROUS: Exported without permission checks -->
<service 
    android:name=".SensitiveService"
    android:exported="true" />
```

**Secure Manifest Configuration:**

```xml
<!-- Secure service configuration -->
<service 
    android:name=".SecureService"
    android:exported="false"
    android:permission="com.example.CUSTOM_PERMISSION" />

<!-- Define custom signature-level permission -->
<permission 
    android:name="com.example.CUSTOM_PERMISSION"
    android:label="Access Secure Service"
    android:description="Allows access to secure service functionality"
    android:protectionLevel="signature" />
```

**Secure Service Implementation:**

```java
public class SecureService extends Service {
    private static final String REQUIRED_PERMISSION = "com.example.CUSTOM_PERMISSION";
    private static final String[] TRUSTED_PACKAGES = {
        "com.example.trustedapp1",
        "com.example.trustedapp2"
    };
    
    @Override
    public IBinder onBind(Intent intent) {
        // Check calling permission
        if (checkCallingPermission(REQUIRED_PERMISSION) != PackageManager.PERMISSION_GRANTED) {
            throw new SecurityException("Permission denied: " + REQUIRED_PERMISSION);
        }
        
        // Additional package validation
        if (!isCallerTrusted()) {
            throw new SecurityException("Untrusted caller");
        }
        
        return binder;
    }
    
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        // Validate calling package
        if (!isCallerTrusted()) {
            Log.w(TAG, "Unauthorized access attempt from: " + getCallingPackage());
            stopSelf();
            return START_NOT_STICKY;
        }
        
        // Perform sensitive operations
        performSensitiveTask();
        return START_STICKY;
    }
    
    private boolean isCallerTrusted() {
        String callingPackage = getPackageManager().getNameForUid(Binder.getCallingUid());
        
        // Verify package signature for additional security
        if (callingPackage != null) {
            try {
                PackageInfo callerInfo = getPackageManager()
                    .getPackageInfo(callingPackage, PackageManager.GET_SIGNATURES);
                return verifySignature(callerInfo.signatures);
            } catch (PackageManager.NameNotFoundException e) {
                return false;
            }
        }
        
        return Arrays.asList(TRUSTED_PACKAGES).contains(callingPackage);
    }
    
    private boolean verifySignature(Signature[] signatures) {
        // Implement signature verification logic
        // Compare with expected signature hashes
        return true; // Simplified for brevity
    }
    
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```

### 3. Implicit Intent Hijacking

> [!danger] Intent Interception Malicious applications can intercept implicit intents meant for legitimate services by registering matching intent filters.

**Vulnerable Code:**

```java
// DANGEROUS: Using implicit intent
Intent serviceIntent = new Intent("com.example.ACTION_PROCESS_DATA");
serviceIntent.putExtra("sensitive_token", userToken);
startService(serviceIntent);
```

**Secure Implementation:**

```java
public class SecureIntentService extends Service {
    
    // Start service securely from client
    public static void startSecurely(Context context, String data) {
        Intent explicit = new Intent(context, SecureIntentService.class);
        // Make intent explicit
        explicit.setPackage(context.getPackageName());
        explicit.putExtra("data", data);
        
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            context.startForegroundService(explicit);
        } else {
            context.startService(explicit);
        }
    }
    
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        // Validate intent source
        if (!isIntentFromTrustedSource(intent)) {
            Log.w(TAG, "Received intent from untrusted source");
            stopSelf();
            return START_NOT_STICKY;
        }
        
        String data = intent.getStringExtra("data");
        if (data != null) {
            processData(data);
        }
        
        return START_NOT_STICKY;
    }
    
    private boolean isIntentFromTrustedSource(Intent intent) {
        String callingPackage = getPackageManager().getNameForUid(Binder.getCallingUid());
        return getPackageName().equals(callingPackage);
    }
    
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```

### 4. Pending Intent Vulnerabilities

> [!danger] Mutable PendingIntents Mutable `PendingIntent` objects can be modified by malicious applications to gain access to non-exported components.

**Vulnerable PendingIntent Creation:**

```java
// DANGEROUS: Mutable PendingIntent
Intent intent = new Intent(this, PrivateActivity.class);
PendingIntent pendingIntent = PendingIntent.getActivity(
    this, 0, intent, PendingIntent.FLAG_UPDATE_CURRENT);
```

**Secure PendingIntent Implementation:**

```java
public class SecurePendingIntentService extends Service {
    
    private PendingIntent createSecurePendingIntent() {
        Intent intent = new Intent(this, MainActivity.class);
        intent.setAction("com.example.SECURE_ACTION");
        
        // Make PendingIntent immutable (API 23+)
        int flags = PendingIntent.FLAG_UPDATE_CURRENT;
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            flags |= PendingIntent.FLAG_IMMUTABLE;
        }
        
        // Use FLAG_ONE_SHOT if should only be triggered once
        if (shouldBeOneShot()) {
            flags |= PendingIntent.FLAG_ONE_SHOT;
        }
        
        return PendingIntent.getActivity(this, 0, intent, flags);
    }
    
    private void handlePendingIntent(PendingIntent receivedIntent) {
        // NEVER trust PendingIntent creator information for authorization
        // DO NOT use: PendingIntent.getCreatorPackage()
        // DO NOT use: PendingIntent.getCreatorUid()
        
        // Instead, use alternative authentication methods
        String callingPackage = getPackageManager().getNameForUid(Binder.getCallingUid());
        if (!isAuthorizedCaller(callingPackage)) {
            throw new SecurityException("Unauthorized PendingIntent usage");
        }
        
        // Process the PendingIntent safely
        try {
            receivedIntent.send();
        } catch (PendingIntent.CanceledException e) {
            Log.e(TAG, "PendingIntent was cancelled", e);
        }
    }
    
    private boolean shouldBeOneShot() {
        // Determine if this PendingIntent should only be used once
        return true;
    }
    
    private boolean isAuthorizedCaller(String packageName) {
        // Verify caller through alternative means
        return verifyCallerSignature(packageName);
    }
    
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```

### 5. Memory Corruption Vulnerabilities

> [!danger] Deserialization Attacks Services processing untrusted data through deserialization or insecure JSON parsing can be vulnerable to memory corruption.

**Vulnerable Code:**

```java
// DANGEROUS: Unsafe deserialization
@Override
public int onStartCommand(Intent intent, int flags, int startId) {
    byte[] serializedData = intent.getByteArrayExtra("user_data");
    if (serializedData != null) {
        try {
            ObjectInputStream ois = new ObjectInputStream(
                new ByteArrayInputStream(serializedData));
            Object userObject = ois.readObject(); // DANGEROUS!
            processUserObject(userObject);
        } catch (Exception e) {
            Log.e(TAG, "Deserialization failed", e);
        }
    }
    return START_NOT_STICKY;
}
```

**Secure Implementation:**

```java
public class SecureDeserializationService extends Service {
    private static final Set<String> ALLOWED_CLASSES = new HashSet<>(Arrays.asList(
        "com.example.SafeDataClass",
        "com.example.UserPreferences"
    ));
    
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        String jsonData = intent.getStringExtra("json_data");
        if (jsonData != null) {
            processJsonSafely(jsonData);
        }
        
        Bundle bundle = intent.getBundleExtra("bundle_data");
        if (bundle != null) {
            processBundleSafely(bundle);
        }
        
        return START_NOT_STICKY;
    }
    
    private void processJsonSafely(String jsonData) {
        try {
            // Use safe JSON parsing - avoid dynamic object creation
            JSONObject json = new JSONObject(jsonData);
            
            // Extract only expected fields
            String name = json.optString("name", "");
            int age = json.optInt("age", 0);
            
            if (isValidInput(name, age)) {
                // Create objects manually instead of automatic deserialization
                UserData userData = new UserData(name, age);
                processUserData(userData);
            }
        } catch (JSONException e) {
            Log.w(TAG, "Invalid JSON data received");
        }
    }
    
    private void processBundleSafely(Bundle bundle) {
        // Avoid using Parcelable classes that might contain native pointers
        // Extract primitive types only
        String safeString = bundle.getString("safe_string");
        int safeInt = bundle.getInt("safe_int", 0);
        
        if (isValidBundleData(safeString, safeInt)) {
            processValidatedData(safeString, safeInt);
        }
    }
    
    private boolean isValidInput(String name, int age) {
        return name != null && name.length() <= 100 && age >= 0 && age <= 150;
    }
    
    private boolean isValidBundleData(String str, int value) {
        return str != null && str.length() <= 1000 && value >= 0;
    }
    
    // Custom ObjectInputStream with class filtering
    private static class SafeObjectInputStream extends ObjectInputStream {
        public SafeObjectInputStream(InputStream in) throws IOException {
            super(in);
        }
        
        @Override
        protected Class<?> resolveClass(ObjectStreamClass desc) throws IOException, ClassNotFoundException {
            String className = desc.getName();
            if (!ALLOWED_CLASSES.contains(className)) {
                throw new SecurityException("Disallowed class: " + className);
            }
            return super.resolveClass(desc);
        }
    }
    
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```

### 6. SQL Injection in Service Database Operations

> [!danger] Database Vulnerabilities Services interacting with databases can be vulnerable to SQL injection if they construct queries insecurely.

**Vulnerable Database Service:**

```java
// DANGEROUS: SQL injection vulnerable
public class VulnerableDatabaseService extends Service {
    private SQLiteDatabase database;
    
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        String userId = intent.getStringExtra("user_id");
        String query = "SELECT * FROM users WHERE id = '" + userId + "'"; // DANGEROUS!
        
        Cursor cursor = database.rawQuery(query, null);
        // Process results...
        
        return START_NOT_STICKY;
    }
}
```

**Secure Database Implementation:**

```java
public class SecureDatabaseService extends Service {
    private SQLiteDatabase database;
    private SQLiteQueryBuilder queryBuilder;
    
    @Override
    public void onCreate() {
        super.onCreate();
        initializeDatabase();
        setupSecureQueryBuilder();
    }
    
    private void setupSecureQueryBuilder() {
        queryBuilder = new SQLiteQueryBuilder();
        queryBuilder.setStrict(true); // Enable strict mode
        queryBuilder.setStrictColumns(true); // Validate columns
        queryBuilder.setStrictGrammar(true); // Limit subqueries
    }
    
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        String userId = intent.getStringExtra("user_id");
        if (!isValidUserId(userId)) {
            Log.w(TAG, "Invalid user ID received");
            return START_NOT_STICKY;
        }
        
        // Use parameterized queries
        String[] selectionArgs = {userId};
        Cursor cursor = database.query(
            "users",                    // table
            null,                       // columns
            "id = ?",                   // selection with placeholder
            selectionArgs,              // selection args
            null,                       // groupBy
            null,                       // having
            null                        // orderBy
        );
        
        processResults(cursor);
        return START_NOT_STICKY;
    }
    
    // Alternative using PreparedStatement
    private void queryWithPreparedStatement(String userId) {
        String sql = "SELECT * FROM users WHERE id = ?";
        try (SQLiteStatement statement = database.compileStatement(sql)) {
            statement.bindString(1, userId);
            // Execute query safely
        }
    }
    
    // Using secure QueryBuilder
    private Cursor secureQuery(String userId, String column) {
        Map<String, String> columnMap = new HashMap<>();
        columnMap.put("user_id", "id");
        columnMap.put("user_name", "name");
        
        queryBuilder.setProjectionMap(columnMap);
        queryBuilder.setTables("users");
        
        return queryBuilder.query(
            database,
            new String[]{column},  // projection
            "id = ?",              // selection
            new String[]{userId},  // selectionArgs
            null,                  // groupBy
            null,                  // having
            null                   // sortOrder
        );
    }
    
    private boolean isValidUserId(String userId) {
        return userId != null && userId.matches("^[0-9]+$") && userId.length() <= 10;
    }
    
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```

### 7. XML External Entities (XXE) Injection

> [!danger] XXE Vulnerabilities Services processing XML input may be vulnerable to XXE attacks if the XML parser is not securely configured.

**Vulnerable XML Processing:**

```java
// DANGEROUS: XXE vulnerable XML parsing
public class VulnerableXMLService extends Service {
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        String xmlData = intent.getStringExtra("xml_data");
        if (xmlData != null) {
            try {
                DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
                DocumentBuilder builder = factory.newDocumentBuilder(); // DANGEROUS!
                Document doc = builder.parse(new ByteArrayInputStream(xmlData.getBytes()));
                processXMLDocument(doc);
            } catch (Exception e) {
                Log.e(TAG, "XML parsing failed", e);
            }
        }
        return START_NOT_STICKY;
    }
}
```

**Secure XML Processing:**

```java
public class SecureXMLService extends Service {
    
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        String xmlData = intent.getStringExtra("xml_data");
        if (xmlData != null && isValidXMLInput(xmlData)) {
            parseXMLSecurely(xmlData);
        }
        return START_NOT_STICKY;
    }
    
    private void parseXMLSecurely(String xmlData) {
        try {
            // Method 1: Using DocumentBuilderFactory with security features disabled
            DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
            
            // Disable DTDs completely
            factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
            
            // Disable external DTDs and stylesheets
            factory.setFeature("http://xml.org/sax/features/external-general-entities", false);
            factory.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
            
            // Disable XInclude processing
            factory.setXIncludeAware(false);
            
            // Disable expansion of entity reference nodes
            factory.setExpandEntityReferences(false);
            
            DocumentBuilder builder = factory.newDocumentBuilder();
            Document doc = builder.parse(new ByteArrayInputStream(xmlData.getBytes()));
            processXMLDocument(doc);
            
        } catch (Exception e) {
            Log.e(TAG, "Secure XML
```