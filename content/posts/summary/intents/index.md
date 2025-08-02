+++
date = '2025-07-30T00:55:36+03:00'
draft = false
title = 'What is your (Intent)ion'
+++

## Summary

>A way to express what you want from specific/special  android components, sometimes you want to give it an *ACTION* or put some *Extra data* that it can read and interpret (understand) and don't forget to that you can *set class* that you cant to talk to wither its in same app class or different app  hence why it's named intention i guess XD .

>[!Who can you send it to?]
>1. Activities: Sent to start new activities within your app or other apps installed on the user's device.
>2. Services: Communicate with background services running in your app or other apps.
>3. Broadcast Receivers: Intents can be broadcast by the system (or an app) and received by appropriate components registered to handle those intents.
>4. Content Providers: Used to interact with content providers, which manage shared data between applications.
>5. Fragments: Intents can be sent to start new fragments within your app.
>6. Custom Components: You can create custom components (e.g., views, widgets) that respond to specific intent actions and categories

## What intent looks like 

### Key Attributes of an Intent

An Intent object consists of several arguments that describe its purpose:
```java
        Intent intent = new Intent();
        
        intent.setAction("android.intent.action.MAIN");
        intent.setData(Uri.parse("content://contacts/people/1"));
        intent.setType("text/plain");
        intent.addCategory("android.intent.category.LAUNCHER");
        intent.setComponent(new ComponentName("com.example.package", "com.example.package.MainActivity"));
        intent.setClass();
        intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
        intent.putExtra("EXTRA_KEY", "Sample Value");
        intent.putExtra("ANOTHER_KEY", 42);
```

> [!note] Important Categories
> 
> - `DEFAULT` category allows your app to respond to implicit intents
> - Without `DEFAULT`, the activity can only be started with explicit component names


>[!Warning]
>You can only sent to any component (Implicit or Explicit) intent if it's defined in the manifest file that exported its set to True  

### Types of  Intent
#### 1. Explicit

_Sending intent to a specific component_
- specify the exact  **Package** (if in different app)& full **class name** of target and using
	- `setComponent()`  or  `setClass()`

```java
Intent intent = new Intent(); 
intent.setComponent(new ComponentName("com.example", "com.example.MyActivity")); 

//or

intent.setClass("com.example", "com.example.MyActivity")
startActivity(intent);
```

#### 2. Implicit

_when you don't know exactly what component you want to interact with, and Android will handle the routing based on the intent Action_

**does not fully specify the target component** (by package and class name). Instead, it contains some data, often just an action (like `ACTION_SEND`), which allows the Android system to find a matching component to handle the intent (through filters). If multiple matching components are found, the user might be presented with a selection dialog

- Using `setAction()` for setting an action string.

```java
// Setting action
Intent intent = new Intent(); 
intent.setAction("com.example.ACTION"); 
startActivity(intent);
```

**Corresponding Intent Filter in Manifest:**

```xml
<intent-filter>
    <action android:name="com.example.ACTION" />
    <category android:name="android.intent.category.DEFAULT" />
</intent-filter>
```

##### Risks (Implicit intent hijacking):

vulnerability where a malicious application can intercept and potentially take over an implicit intent meant for another application, this allows a malicious application to register an intent filter to intercept the intent instead of the intended application.

Depending on the intent content, attackers could read or modify sensitive information or interact with mutable objects, such as [mutable](https://developer.android.com/reference/android/app/PendingIntent#FLAG_MUTABLE) [PendingIntents](https://developer.android.com/reference/android/app/PendingIntent) or [Binders](https://developer.android.com/reference/android/os/Binder).

##### Mitigation :

Unless the application requires it, make intents explicit by calling `setPackage()`. This allows only by a specific component . preventing untrusted applications from intercepting the data sent along with the intent. The following snippet shows how to make an intent explicit:

```java
Intent intent = new Intent("android.intent.action.CREATE_DOCUMENT");
intent.addCategory("android.intent.category.OPENABLE");
intent.setPackage("com.some.packagename");
intent.setType("*/*");
intent.putExtra("android.intent.extra.LOCAL_ONLY", true);
intent.putExtra("android.intent.extra.TITLE", "Some Title");
startActivity(intent);
```

for more go to [[Implicit intent hijacking]]
more attacks 
### Intent  Attacks & Mitigation

#### Implicit Intent Hijacking

> [!danger] Malicious applications can intercept implicit intents by registering matching intent filters, allowing them to read or modify sensitive information.

**Attack Scenarios:**

1. **Insecure Broadcasts**
    - Messaging apps broadcasting implicitly can be intercepted
    - All implicit broadcasts are delivered to every registered receiver across all apps
2. **Insecure Activity Launches**
    - Banking apps sharing credit card details via implicit intents
    - Malicious apps can manipulate their position in Activity Chooser using `android:priority`
3. **Return Value Attacks (`startActivityForResult()`)**
    - Intercepting apps can use `setResult()` to return crafted data
    - Commonly leads to **arbitrary file/image theft**
    - Attackers return URIs pointing to private directory files
4. **File Overwrite Attacks**
    - Path-traversal characters in filenames (e.g., `../lib.so`)
    - Can lead to **arbitrary code execution** if files are written to critical locations

 The attacker can control the position of his app in the list using the `android:priority="num"` attribute in the `<intent-filter>` declaration The attacker can thus intercept credit intent as follows

```xml
<activity android:name=".EvilActivity">
    <intent-filter android:priority="999">
        <action android:name="com.victim.ADD_CARD_ACTION" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

#### Intent Redirection

> [!danger]  One app component receives an intent and forwards it to another component without proper validation.

**Vulnerable Pattern:**

- **Attack Activity** (malicious app) → **Proxy Activity** (vulnerable app) → **Protected Target Activity**

**Common Vulnerability:**

```java
// Vulnerable code - no validation
Parcelable extraIntent = getIntent().getParcelableExtra("extra_intent");
if (extraIntent instanceof Intent) {
    startActivity((Intent) extraIntent); // Dangerous!
}
```

OR

```java
Intent intent = getIntent();
Intent nextIntent = (Intent) intent.getParcelableExtra("nextIntent");
if (this.nextIntent.getStringExtra("reason").equals("next")) {
            startActivity(this.nextIntent);
        }
```

**That can lead to :**

- **Open Redirect** - Malicious URLs routed through proxy
- **Local File Inclusion (LFI)** - Access to local files if WebView has universal access enabled
- **Protected FileProvider Access** - Access non-exported ContentProviders

#### mitigation
##### 1. Validate Intent Destinations

**verify where intents are being redirected** before forwarding them:

```java
Intent intent = getIntent();
// Get the component name of the nested intent
Intent forward = (Intent) intent.getParcelableExtra("key");
ComponentName name = forward.resolveActivity(getPackageManager());

// Check that the package name and class name contain the expected values
if (name.getPackageName().equals("safe_package") &&
    name.getClassName().equals("safe_class")) {
    // Redirect the nested intent
    startActivity(forward);
}
```

##### 2. Use Pending Intent Objects

**Pending Intents prevent component export** and make target actions immutable to send intents that can't be tampered with 

```java
// Create immutable PendingIntent
PendingIntent pendingIntent = PendingIntent.getActivity(
    getContext(), 
    0, 
    new Intent(intentAction), 
    PendingIntent.FLAG_IMMUTABLE
);

// Ensure explicit component targeting
Intent intent = new Intent(intentAction);
intent.setClassName(packageName, className);
PendingIntent pendingIntent = PendingIntent.getActivity(
    getContext(), 
    0, 
    intent, 
    PendingIntent.FLAG_IMMUTABLE
);
```

##### 3. Implement IntentSanitizer

Use **IntentSanitizer** to create sanitized copies of intents with allowlisted components and data:
```java
Intent sanitizedIntent = new IntentSanitizer.Builder()
    .allowComponent("com.example.ActivityA")
    .allowData("com.example")
    .allowType("text/plain")
    .build()
    .sanitizeByThrowing(intent);
```

##### 4. Explicit Intent Usage

**Use explicit intents** for sensitive operations to control component targeting:

```java
Intent intent = new Intent("android.intent.action.CREATE_DOCUMENT");
intent.addCategory("android.intent.category.OPENABLE");
intent.setPackage("com.some.packagename"); // Explicitly set package
intent.setType("*/*");
startActivity(intent);
```

##### 5. [Dynamic determination of intent receivers](https://blog.oversecured.com/Interception-of-Android-implicit-intents/#dynamic-determination-of-intent-receivers)
Some apps try to stop the activity picker appearing by automatically determining a single recipient and setting it in the intent settings (which is also very common when launching services, since implicit intents are forbidden in service launch)

```java
Intent intent = new Intent("com.victim.ADD_CARD_ACTION");
intent.putExtra("credit_card_number", num.getText().toString());
intent.putExtra("holder_name", name.getText().toString());
//...
for (ResolveInfo info : getPackageManager().queryIntentActivities(intent, 0)) {
    intent.setClassName(info.activityInfo.packageName, info.activityInfo.name);
    startActivity(intent);
    return;
}
```

this is more advantageous, because it eliminates the need for user interaction and automatically specifies the attacker’s activity

##### 6. **Minimize component exposure** and implement proper permissions:

```xml
<!-- Avoid exporting components unless necessary -->
<activity android:name=".MyActivity" android:exported="false"/>
```

```java
// Enforce permissions at code level
public class MyExportService extends Service {
    @Override
    public IBinder onBind(Intent intent) {
        enforceCallingPermission(Manifest.permission.READ_CONTACTS,
            "Calling app doesn't have READ_CONTACTS permission.");
        return binder;
    }
}
```


## Receiving Intent Results
### getIntent()
```java
Intent recivedIntent = getIntent();  
String action = recivedIntent.getAction();
// now access it 
  
//to return a result  when using onActivityResult 
recivedIntent.putExtra("token",1094795585);  
setResult(RESULT_OK, recivedIntent);  
finish(); 
```


### Legacy: onActivityResult()

```java
Intent intent = new Intent(this, MessageActivity.class);
intent.putExtra(EXTRA_MESSAGE, message);
startActivityForResult(intent, ACTIVITY_ID01);

@Override 
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
    if (requestCode == ACTIVITY_ID01) {
        // Process data from activity 1
        if (resultCode == RESULT_OK && data != null) {
            String token = data.getStringExtra("token");
            // Handle result
        }
    }
}
```

### Modern: registerForActivityResult()

```java
ActivityResultLauncher<Intent> activityLauncher = 
    registerForActivityResult(
        new ActivityResultContracts.StartActivityForResult(),
        result -> {
            if (result.getResultCode() == RESULT_OK) {
                Intent data = result.getData();
                if (data != null) {
                    // Process result data
                    String token = data.getStringExtra("token");
                }
            }
        }
    );

// Launch activity
Intent intent = new Intent(this, TargetActivity.class);
activityLauncher.launch(intent);
```

### Returning Results

```java
// In the called activity
Intent resultIntent = getIntent();
resultIntent.putExtra("token", "1094795585");
setResult(RESULT_OK, resultIntent);
finish();
```

##  PendingIntents

> [!info] Definition: A **PendingIntent** allows one application to grant another application permission to execute a predefined action on its behalf, even if the creator app is no longer running.

It's often used in scenarios where you want to allow an action to be performed in the future, even if your application is not running.

Hence PendingIntent can be used with different components using methods like :

1. `PendingIntent.getActivity()` : Retrieve a PendingIntent to start an Activity
2. `PendingIntent.getBroadcast()` : Retrieve a PendingIntent to perform a Broadcast
3. `PendingIntent.getService()` : Retrieve a PendingIntent to start a Service


> [!example] Example
> Imagine that you have a **non exported activity A** which can be launched from your own application’s activities. You want though other applications to be able to call A under specific conditions. This is where the `pending intent` comes to place. The idea is to **wrap a normal intent** (base intent) that you would use to start activity A into an object, send this object to the other application and let the other application to unwrap the intent and send it back to activity A.  _Dimitrios Valsamaras (+Chopin) : Pending Intents: A Pentester’s view_

**Code Example:**  

You want to **show a notification**, and when the user clicks it, it opens a **NonExportedActivity** within your app using a PendingIntent.

```java
// Step 1: Create base Intent for non-exported activity
Intent intent = new Intent(this, NonExportedActivity.class);
intent.putExtra("msg", "Hello from PendingIntent!");

// Step 2: Wrap Intent in PendingIntent with security flags
PendingIntent pendingIntent = PendingIntent.getActivity(
    this,                              // Context
    0,                                 // requestCode
    intent,                           // Base Intent
    PendingIntent.FLAG_IMMUTABLE      // Security flag
);

// Step 3: Use in notification
NotificationCompat.Builder builder = new NotificationCompat.Builder(this, channelId)
    .setSmallIcon(R.drawable.ic_launcher_foreground)
    .setContentTitle("PendingIntent Demo")
    .setContentText("Click to launch NonExportedActivity")
    .setAutoCancel(true)
    .setContentIntent(pendingIntent);   // Attach PendingIntent

notificationManager.notify(1, builder.build());
```

**NonExportedActivity.java**
```java
	String msg = getIntent().getStringExtra("msg");

    TextView textView = findViewById(R.id.textView);
    textView.setText(msg);
```

**PendingIntent Flags**

| Flag             | Value    | Description                             |
| ---------------- | -------- | --------------------------------------- |
| `FLAG_MUTABLE`   | 33554432 | Allows intent data/extras to be changed |
| `FLAG_IMMUTABLE` | 67108864 | Locks intent - recommended for security |
| `FLAG_ONE_SHOT`  | -        | Prevents replay attacks                 |
|                  |          |                                         |

### How to create a pending intent

Each section focuses on a specific functionality or use case, making the code easier to understand and maintain. The methods are separated logically, and related functionality is grouped together.

#### 1. Creating Activity PendingIntent
```java
// Method to create a PendingIntent for starting an Activity
private PendingIntent createActivityPendingIntent() {
    // Step 1: Create the base Intent with explicit component
    Intent baseIntent = new Intent("My.Action");
    baseIntent.setClassName("com.example.myapp", "com.example.myapp.TargetActivity");
    
    // Step 2: Create PendingIntent with FLAG_IMMUTABLE for security
    return PendingIntent.getActivity(
        this,                    // Context
        0,                       // Request code
        baseIntent,              // Base Intent
        PendingIntent.FLAG_IMMUTABLE  // Security flag
    );
}
```

#### 2. Creating Service PendingIntent
```java
// Method to create a PendingIntent for a Service
private PendingIntent createServicePendingIntent() {
    Intent serviceIntent = new Intent("My.Service.Action");
    serviceIntent.setClassName("com.example.myapp", "com.example.myapp.MyService");
    
    return PendingIntent.getService(
        this,
        0,
        serviceIntent,
        PendingIntent.FLAG_IMMUTABLE
    );
}
```

#### 3. Creating Broadcast PendingIntent
```java
// Method to create a PendingIntent for a Broadcast
private PendingIntent createBroadcastPendingIntent() {
    Intent broadcastIntent = new Intent("My.Broadcast.Action");
    broadcastIntent.setClassName("com.example.myapp", "com.example.myapp.MyBroadcastReceiver");
    
    return PendingIntent.getBroadcast(
        this,
        0,
        broadcastIntent,
        PendingIntent.FLAG_IMMUTABLE
    );
}
```

#### 4. Creating One-Shot PendingIntent
```java
// Method to create a one-shot PendingIntent (can only be used once)
private PendingIntent createOneShotPendingIntent() {
    Intent intent = new Intent("My.OneShot.Action");
    intent.setClassName("com.example.myapp", "com.example.myapp.OneShotActivity");
    
    return PendingIntent.getActivity(
        this,
        0,
        intent,
        PendingIntent.FLAG_IMMUTABLE | PendingIntent.FLAG_ONE_SHOT
    );
}
```

#### 5. Sending PendingIntent to Another App
```java
// Method to send PendingIntent to another app
private void sendPendingIntentToAnotherApp() {
    // Create the PendingIntent
    PendingIntent pendingIntent = createActivityPendingIntent();
    
    // Create Intent to target another app
    Intent targetAppIntent = new Intent();
    targetAppIntent.setClassName("com.example.otherapp", "com.example.otherapp.ReceiverActivity");
    
    // Add PendingIntent as extra
    targetAppIntent.putExtra("pendingIntent", pendingIntent);
    
    // Start the other app's activity
    startActivity(targetAppIntent);
}
```

#### 6. Using PendingIntent with Notification
```java
// Method to use PendingIntent with NotificationManager (common use case)
private void createNotificationWithPendingIntent() {
    // Create PendingIntent
    PendingIntent pendingIntent = createActivityPendingIntent();
    
    // Create notification with PendingIntent
    NotificationCompat.Builder builder = new NotificationCompat.Builder(this, "channel_id")
        .setSmallIcon(R.drawable.ic_notification)
        .setContentTitle("My Notification")
        .setContentText("Click to open activity")
        .setContentIntent(pendingIntent)  // Set the PendingIntent
        .setAutoCancel(true);
        
    NotificationManager notificationManager = 
        (NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE);
    notificationManager.notify(1, builder.build());
}
```

#### 7. Using PendingIntent with AlarmManager
```java
// Method to use PendingIntent with AlarmManager
private void scheduleAlarmWithPendingIntent() {
    // Create PendingIntent for broadcast
    PendingIntent alarmIntent = createBroadcastPendingIntent();
    
    // Schedule alarm
    AlarmManager alarmManager = (AlarmManager) getSystemService(Context.ALARM_SERVICE);
    long triggerTime = System.currentTimeMillis() + 60000; // 1 minute from now
    
    alarmManager.setExact(AlarmManager.RTC_WAKEUP, triggerTime, alarmIntent);
}
```

#### 8. Receiver Activity (Separate Class)
```java
// Example of receiving and using a PendingIntent in another app
public class ReceiverActivity extends AppCompatActivity {
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_receiver);
        
        // Retrieve PendingIntent from the Intent extras
        Intent intent = getIntent();
        PendingIntent receivedPendingIntent = intent.getParcelableExtra("pendingIntent");
        
        if (receivedPendingIntent != null) {
            try {
                // Send/execute the PendingIntent
                receivedPendingIntent.send();
            } catch (PendingIntent.CanceledException e) {
                Log.e("ReceiverActivity", "PendingIntent was cancelled", e);
            }
        }
    }
}
```

#### 9. Mutable PendingIntent Example (Separate Class)
```java
// Example with mutable PendingIntent (use with caution)
public class MutablePendingIntentExample {
    
    private PendingIntent createMutablePendingIntent(Context context) {
        Intent baseIntent = new Intent("My.Mutable.Action");
        baseIntent.setClassName("com.example.myapp", "com.example.myapp.MutableActivity");
        
        // Only use FLAG_MUTABLE when absolutely necessary
        return PendingIntent.getActivity(
            context,
            0,
            baseIntent,
            PendingIntent.FLAG_MUTABLE  // Use with extreme caution
        );
    }
}
```

### Pending Intent  Attacks & Mitigations 

#### Mutable PendingIntents

> [!danger] By default, PendingIntents were historically mutable (until Android R). Malicious apps can modify the inner Intent, potentially accessing non-exported components.

**Attack Scenarios:**

1. **Notification Hijack**
    - Malicious app with notification listener permission fetches PendingIntent
    - Modifies Intent to launch attacker's activity with victim's permissions
2. **Permission Escalation**
    - App A has `READ_CONTACTS` permission, creates mutable PendingIntent
    - App B modifies Intent to read contacts without having the permission

#### Replay Attacks

> [!danger] Risk  
> PendingIntents can be replayed unless `FLAG_ONE_SHOT` is set, allowing malicious apps to repeat actions multiple times.

#### Mitigation 
##### 1. Set Explicit Components
```java
Intent intent = new Intent(intentAction);
// Explicitly set component to prevent redirection
intent.setClassName(packageName, className);

PendingIntent pendingIntent = PendingIntent.getActivity(
    getContext(),
    0,
    intent,
    PendingIntent.FLAG_IMMUTABLE
);
```

##### 2. Use FLAG_IMMUTABLE

If your app targets Android 6 (API level 23) or higher, [specify mutability](https://developer.android.com/guide/components/intents-filters#DeclareMutabilityPendingIntent). 
```java
// For Android 6+ (API 23), strongly recommended
// For Android 11+ (API 30), mutability must be specified
PendingIntent pendingIntent = PendingIntent.getActivity(
    getContext(),
    0,
    new Intent(intentAction),
    PendingIntent.FLAG_IMMUTABLE
);
```

##### 3. Prevent Replay Attacks
```java
// For one-time actions
PendingIntent pendingIntent = PendingIntent.getActivity(
    getContext(),
    0,
    new Intent(intentAction),
    PendingIntent.FLAG_IMMUTABLE | PendingIntent.FLAG_ONE_SHOT
);
```

> [!warning] Critical Security Rule **Always ensure the base Intent wrapped in PendingIntent has the component name explicitly set** to one of your own components.

## Deep Links

>[!info] A deep link is ==a special type of hyperlink that allows users to navigate directly to a specific page or content within an app, bypassing the app's homepage==. This is done by declaring what the component will accepts through it's filters 

### How Deep Links Work

![[Pasted image 20250713181259.png]]

For an application to handle deep links it requires:

1. **Intent Filter Required** - Activity must have intent filter in `AndroidManifest.xml`
2. **Exported Activity** - Must be explicitly exported (`android:exported="true"`)
3. **URI Scheme Definition** - Specify scheme, host, and path in `<data>` tag

In Details:
1. The `action` need to be `action.VIEW`

2. Category 
	- `DEFALUT` : to handle implicit intents without specifying the component name
	- `BROWSABLE` : allow browsers to open the link in your app.

3. Data:
	- Define the URI format with at least `android:scheme`
	- Optionally use `android:path`, `pathPrefix`, or `pathPattern` to refine which URIs open this activity.
	- (if no data is set ) : The Chrome `intent:` scheme solves this by allowing a site to create explicit intents 

Chrome on Android implements a [custom scheme `intent:`](https://developer.chrome.com/docs/android/intents) with a lot more features than the regular deep links. Which make them very useful for an attacker.

`intent:` features:

1. It's a generic intent
2. Control the action and category
3. Target specific app and class
4. Add extra values

### Deep Link Manifest Example

> [!warning] Intent Filter Separation Create separate intent filters for different URL schemes. Multiple `<data>` elements in the same filter can create unintended combinations.

The following XML snippet shows an intent filter deep linking. The URIs `“example://gizmos”` and `“http://www.example.com/gizmos”` both resolve to this activity.

```xml
<activity
    android:name="com.example.android.GizmosActivity"
    android:label="@string/title_gizmos"
    android:exported="true">
    
    <intent-filter android:label="@string/filter_view_http_gizmos">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <!-- HTTP/HTTPS deep link -->
        <data android:scheme="http"
              android:host="www.example.com"
              android:pathPrefix="/gizmos" />
    </intent-filter>
    
    <intent-filter android:label="@string/filter_view_example_gizmos">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <!-- Custom scheme deep link -->
        <data android:scheme="example"
              android:host="gizmos" />
    </intent-filter>
</activity>
```

But what will happen if we didn't separate the data?
Although it's possible to include multiple `<data>` elements in the same filter, it's important that you create separate filters when your intention is to declare unique URLs

**Problem Example:**
```xml
<!-- DON'T DO THIS - Creates unintended combinations -->
<intent-filter>
    <data android:scheme="https" android:host="www.example.com" />
    <data android:scheme="app" android:host="open.my.app" />
</intent-filter>
<!-- This accepts: https://www.example.com, app://open.my.app -->
<!-- But also: app://www.example.com, https://open.my.app -->
```

It might seem as though this supports only `https://www.example.com` and `app://open.my.app`. However, it actually supports those two, plus these: `app://www.example.com` and `https://open.my.app` thus adding more attack vectors 


### Types of Deep Links

#### 1. General Deep Links

- **Definition:** URIs of any scheme (`geo:`, `example://`, `vaadata://`)
- **Behavior:** May trigger disambiguation dialog if multiple apps can handle the URI
- **Implementation:** Requires standard intent filter with `VIEW` action, `DEFAULT` and `BROWSABLE` categories

#### 2. Web Links

- **Definition:** Deep links specifically using **HTTP and HTTPS schemes**
- **Behavior:**
    - **Android 12+:** Always show in browser by default unless app is domain-approved
    - **Earlier versions:** May show disambiguation dialog
- **Implementation:** Same as general deep links but with `http`/`https` schemes

#### 3. Android App Links

- **Definition:** Web links with `android:autoVerify="true"` (Android 6.0+ API 23+)
- **Behavior:** Open immediately if app is verified default handler, bypassing disambiguation
- **Requirements:**
    - HTTP/HTTPS schemes only
    - Domain ownership verification required
    - `assetlinks.json` file on web server

### Android App Links Implementation

#### Step 1: Create Intent Filters

```xml
<intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="https"
          android:host="yourdomain.com" />
</intent-filter>
```

#### Step 2: Digital Asset Links File

Host `assetlinks.json` at `https://yourdomain.com/.well-known/assetlinks.json`:

```json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.example.yourapp",
    "sha256_cert_fingerprints": ["14:6D:E9:83:C5:73:06:50:D8:EE:B9:95:2F:34:FC:64:16:A0:83:42:E6:1D:BE:A8:8A:04:96:B2:3F:CF:44:E5"]
  }
}]
```


### Reading Deep Link Data

```java
@Override
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.main);
    
    Intent intent = getIntent();
    String action = intent.getAction();
    Uri data = intent.getData();
    
    // Use 'data' and 'action' to render appropriate content
    if (Intent.ACTION_VIEW.equals(action) && data != null) {
        // Handle deep link data
        String scheme = data.getScheme();
        String host = data.getHost();
        String path = data.getPath();
        // Process the URI safely
    }
}
```


#### Risk & Impact

>The security risks associated with deep links stem from their core capability of enabling seamless navigation and interaction within mobile applications. Deep link vulnerabilities arise from weaknesses in the implementation or handling of deep links.

They can be hijacked like any other implicit intent if the `<intent-filter>` are properly setup

_The lack of a proper deep link validation mechanism, or the unsafe use of deeplinks, can aid malicious users in performing attacks such as host validation bypass, cross-app scripting, and remote code execution within the permissions context of the vulnerable application. Depending on the nature of the application, this can result in unauthorized access to sensitive data or functions._ (Developer Docs)

An example is some in-app navigation that uses deep links which expose and increase attack surface  

### Deep Link Vulnerabilities & Attacks

>[!warning] The lack of a proper deep link validation mechanism, or the unsafe use of deeplinks, can aid malicious users in performing attacks such as host validation bypass, cross-app scripting, and remote code execution within the permissions context of the vulnerable application. Depending on the nature of the application, this can result in unauthorized access to sensitive data or functions. _(Developer Docs)_  

They can be hijacked like any other implicit intent if the `<intent-filter>` are properly setup

#### Sensitive Data Transfer

> [!danger]  Malicious applications can register intent filters for the same deep links and intercept sensitive data.

**Impact:** Account takeover, credential theft, sensitive information exposure

#### Parameter Injection Attacks

> [!danger] Common Vulnerabilities

1. **Cross-Site Scripting (XSS)**
    
    ```java
    // Vulnerable code
    String url = getIntent().getData().getQueryParameter("url");
    webView.loadUrl(url); // No validation!
    ```
    
2. **Remote Code Execution (RCE)**
    
    ```java
    // Dangerous - parameter passed to exec
    String command = getIntent().getData().getQueryParameter("cmd");
    Runtime.getRuntime().exec(command);
    ```
    
3. **Path Traversal**
    
    ```java
    // Vulnerable to ../../../sensitive_file
    String filename = getIntent().getData().getQueryParameter("file");
    File file = new File("/app/files/" + filename);
    ```
    
#### WebView-Specific Attacks

> [!danger] WebView Vulnerabilities

1. **Arbitrary URL Loading**
    
    - Stealing authentication tokens
    - Loading malicious content
2. **JavaScript Injection**
    
    ```java
    // Vulnerable concatenation
    String userInput = getIntent().getData().getQueryParameter("data");
    webView.loadUrl("javascript:processData('" + userInput + "')");
    ```
    
3. **File Access Attacks**
    
    - `setAllowFileAccessFromFileURLs(true)` enables local file reading
    - `setAllowUniversalAccessFromFileURLs(true)` allows cross-origin requests
4. **JavaScript Interface Exploitation**
    
    ```java
    // Exposing sensitive methods
    webView.addJavascriptInterface(new SensitiveAPI(), "Android");
    ```
    
#### Chrome Intent Scheme Exploitation

Chrome's `intent:` scheme provides more control than regular deep links:

```
intent://example.com/path#Intent;
  action=android.intent.action.VIEW;
  category=android.intent.category.BROWSABLE;
  component=com.victim.app/.VulnerableActivity;
  S.extra_data=malicious_payload;
end;
```

### Deep Link Security Mitigations

#### 1. Use Android App Links

- Requires domain ownership verification
- Prevents other apps from intercepting your links
- Uses only HTTP/HTTPS (no custom schemes)

```xml
<!-- App Links configuration -->
<intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="https" android:host="yourdomain.com" />
</intent-filter>
```

#### 2. Robust Data Validation

```java
public boolean isValidDeepLinkUri(Uri uri) {
    if (uri == null) return false;
    
    // Validate scheme
    String scheme = uri.getScheme();
    if (!"https".equals(scheme)) return false;
    
    // Validate host (avoid common bypasses)
    String host = uri.getHost();
    if (host == null || !host.equals("yourdomain.com")) return false;
    
    // Validate path
    String path = uri.getPath();
    if (path == null || !path.startsWith("/allowed/")) return false;
    
    return true;
}
```

#### 3. WebView Security Configuration

```java
WebSettings settings = webView.getSettings();

// Disable risky features
settings.setGeolocationEnabled(false);
settings.setAllowContentAccess(false);
settings.setAllowFileAccess(false);  // API 29 and lower
settings.setAllowFileAccessFromFileURLs(false);
settings.setAllowUniversalAccessFromFileURLs(false);

// Disable JavaScript if not needed
settings.setJavaScriptEnabled(false);

// Use WebViewAssetLoader for local assets
WebViewAssetLoader assetLoader = new WebViewAssetLoader.Builder()
    .addPathHandler("/assets/", new AssetsPathHandler(this))
    .build();

webView.setWebViewClient(new WebViewClient() {
    @Override
    public WebResourceResponse shouldInterceptRequest(WebView view, WebResourceRequest request) {
        return assetLoader.shouldInterceptRequest(request.getUrl());
    }
});
```

#### 4. Input Sanitization

```java
// Safe parameter handling
public String sanitizeParameter(String input) {
    if (input == null) return "";
    
    // Remove dangerous characters
    input = input.replaceAll("[<>\"'&]", "");
    
    // Validate against whitelist
    if (!input.matches("[a-zA-Z0-9_-]+")) {
        throw new SecurityException("Invalid parameter format");
    }
    
    return input;
}
```

#### 5. Content Provider Security

```java
// Prevent path traversal
public String getSecurePath(String fileName) throws IOException {
    File baseDir = new File("/app/safe_directory/");
    File requestedFile = new File(baseDir, fileName);
    
    // Canonicalize paths to prevent traversal
    String basePath = baseDir.getCanonicalPath();
    String requestedPath = requestedFile.getCanonicalPath();
    
    if (!requestedPath.startsWith(basePath)) {
        throw new SecurityException("Path traversal attempt detected");
    }
    
    return requestedPath;
}
```

#### 6. Safe Browsing Integration

```xml
<!-- Enable Safe Browsing in manifest -->
<application>
    <meta-data
        android:name="android.webkit.WebView.EnableSafeBrowsing"
        android:value="true" />
</application>
```

## Under The hood ( Android Binder)

>[!info] Binder part of android system that allows different Android components to talk to each other safely and efficiently. 

>[!example] Think of it like a messenger passing notes between students in class. The Binder makes sure the messages are secure and can't be read by anyone else. It also makes sure the messages are delivered quickly and efficiently, so the students can focus on learning.

### Intent-to-Binder Translation Process
we can trace how a high-level Intent is translated to low-level Binder communications:

1. Application Code: The `startActivity(intent)` function is called in your application code, requesting that an activity be started.

2. Intent Processing: Security checks are performed on the intent to ensure it adheres to Android's permission system. One example of such a check is the `prepareToLeaveProcess()` method.

3. Service Interaction: If the intent targets a service, the `getService().startActivity(intent)` method forwards the request to that service.

4. Parcel Creation: The intent is serialized into a Parcel object, which acts as a container for the data being sent through Binder.

5. Native Transition: A native method called `writeStrongBinder()` is invoked, transitioning from the application layer to the native interface layer where libbinder (the C++ implementation of Binder) resides.

6. Binder Driver: The kernel handles inter-process communication (IPC) through the /dev/binder driver, which manages Binder's communication at a low level.
7. Target Process: Upon receiving the IPC request, the target process deserializes the Parcel object back into an intent, allowing the receiving app to process the incoming

for more info go to [[Android Binder]]

## Resources & Further Reading

### Official Documentation

- [Android Intents and Intent Filters](https://developer.android.com/guide/components/intents-filters)
- [PendingIntent Documentation](https://developer.android.com/reference/android/app/PendingIntent)
- [App Links Documentation](https://developer.android.com/training/app-links)
- [Deep Linking Guide](https://developer.android.com/training/app-links/deep-linking)
-  [Implicit Intent Hijacking](https://developer.android.com/privacy-and-security/risks/implicit-intent-hijacking)
- [PendingIntent Security](https://developer.android.com/privacy-and-security/risks/pending-intent)
- [Unsafe Deep Link Usage](https://developer.android.com/privacy-and-security/risks/unsafe-use-of-deeplinks)

### Security References

- https://blog.oversecured.com/Interception-of-Android-implicit-intents/

### Advanced Topics

- [WebView Security Best Practices](https://developer.android.com/guide/webapps/webview-security)
- [Content Provider Security](https://developer.android.com/guide/topics/providers/content-provider-security)
- [Android Binder Deep Dive](https://developer.android.com/reference/android/os/Binder)


 intent  
- [Docs implicit hijacking](https://developer.android.com/privacy-and-security/risks/implicit-intent-hijacking)
- [Moving from Android StartActivityForResult to registerForActivityResult](https://medium.com/@steves2001/moving-from-android-startactivityforresult-to-registerforactivityresult-76ca04044ff1)
- [Docs intents result](https://developer.android.com/training/basics/intents/result)
- [StartActivityForResult is deprecated](https://medium.com/realm/startactivityforresult-is-deprecated-82888d149f5d)

Pending intent 
- [DOCS PendingIntent](https://developer.android.com/reference/android/app/PendingIntent)
- https://developer.android.com/guide/components/intents-filters#PendingIntent
- [Pending Intents: A Pentester’s view](https://valsamaras.medium.com/pending-intents-a-pentesters-view-92f305960f03)
- [Hijacking notification ](https://infosecwriteups.com/notification-hijack-how-pendingintent-can-be-exploited-547b0892b7f7)

Deep links
- [Docs deep-linking](https://developer.android.com/training/app-links/deep-linking)
- [Deep links from zero to hero](https://www.youtube.com/watch?v=SCl_rdp0Wik)
- [medium: Introduction to deep links](https://medium.com/androiddevelopers/the-deep-links-crash-course-part-1-introduction-to-deep-links-2189e509e269)
- [unsafe-use-of-deeplinks](https://developer.android.com/privacy-and-security/risks/unsafe-use-of-deeplinks)
