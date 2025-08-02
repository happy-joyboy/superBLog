+++
date = '2025-07-30T00:55:36+03:00'
draft = false
title = 'Android Permissions'
+++

## 1. Android Sandboxing

Android uses application sandboxing through user-based isolation. The system operates on a **principle of least privilege** where each application has its own unique Linux user ID (UID), and must explicitly request access to protected resources. Preventing applications from accessing each other's data without explicit permissions.

### User ID Implementation:

| User Type         | UID Range         | Example                        | Storage Location                                |
| ----------------- | ----------------- | ------------------------------ | ----------------------------------------------- |
| System Users      | 1000-9999         | System (1000), Radio (1001)    | Defined in kernel (android_filesystem_config.h) |
| Application Users | Starting at 10000 | User application (e.g., 10129) | Tracked in /data/system/packages.xml            |

Example filesystem representation:

- `/data/data/u0_a129/` (where 'a129' represents UID 10129)

## 2. Permission Storage and Management

Android permissions are defined and stored in multiple locations:

### System Permission Files:

| File Location                   | Purpose                            | Contents                                    |
| ------------------------------- | ---------------------------------- | ------------------------------------------- |
| `/etc/permissions/platform.xml` | Defines core system permissions    | System UID-permission mappings              |
| `/data/system/packages.xml`     | Tracks installed apps' permissions | App UIDs, granted permissions, certificates |

**Example from `packages.xml`**:

```xml
<package name="com.apphacking.privacy" codePath="..." nativeLibraryPath="...">
  <userId>10129</userId>
  <cert>...</cert>
  <perms>
    <item name="android.permission.READ_CONTACTS" granted="true"/>
    <item name="android.permission.INTERNET" granted="true"/>
  </perms>
</package>
```

## 3. Permission Types

### By Source (who gave it)

| Permission Source  | Definition              | Example                            |
| ------------------ | ----------------------- | ---------------------------------- |
| System Permissions | Pre-defined by Android  | `android.permission.READ_CONTACTS` |
| Custom Permissions | Defined by applications | `com.apphacking.privacy.USER_INFO` |

### By Grant Time (when permission is applied)

#### 3.1 Install-time Permissions
Automatically granted when app is installed - minimal risk to privacy/security.

**Normal Permissions:**

- Cover low-risk operations
- Granted at installation without user interaction
- Examples: `INTERNET`, `VIBRATE`

**Signature Permissions:**

- Granted only to apps signed with same developer certificate
- Used for secure communication between same-developer apps
- Example: Data sharing between messaging and calendar apps from same developer

#### 3.2 Runtime Permissions (Dangerous)
Introduced in Android 6.0 (API level 23) - protect sensitive user data.

**Key Characteristics:**

- Require explicit user approval during app use
- Can be granted or denied by users
- Apps must handle denials gracefully

**Examples:**

- `android.permission.CAMERA`
- `android.permission.ACCESS_FINE_LOCATION`
- `android.permission.READ_CONTACTS`

#### 3.3 Special Permissions
Allow access to powerful system features via "Special app access" settings.

**Examples:**

- `android.permission.SYSTEM_ALERT_WINDOW` - Draw over other apps
- `android.permission.WRITE_SETTINGS` - Modify system settings

## 4. Protection Levels

| Protection Level        | Description                                      | User Experience                               | Risk Level  |
| ----------------------- | ------------------------------------------------ | --------------------------------------------- | ----------- |
| `normal`                | Lower-risk permissions for isolated features     | **Automatically granted** at installation     | Low         |
| `dangerous`             | Higher-risk permissions accessing private data   | **Explicit user consent required** at runtime | Medium-High |
| `signature`             | Only granted to apps with matching certificate   | **Silently granted** if certificate matches   | High        |
| `knownSigner`           | Only granted to apps with allowed certificate    | **Silently granted** if certificate listed    | High        |
| `signature\|privileged` | For matching signature OR privileged system apps | **Silently granted** for system apps          | Very High   |
|                         |                                                  |                                               |             |

> [!note] `signatureOrSystem` is deprecated since API level 23 (Android 6.0). It was previously used to grant permissions to apps that were either signed with the same certificate as the system or were pre-installed system apps.


## 6. Using Permissions

### 6.1 Declaring Permissions
Permissions are declared in `AndroidManifest.xml` using the `<uses-permission>` tag.

**Basic Declaration:**
```xml
<uses-permission android:name="android.permission.CAMERA" />
```

**Version-Specific Declaration:**
```xml
<uses-permission 
    android:name="android.permission.READ_EXTERNAL_STORAGE" 
    android:maxSdkVersion="28" />
```

### 6.2 Checking Permission Status (if the app has it or not)

```java
import androidx.core.content.ContextCompat;
import android.content.pm.PackageManager;

if (ContextCompat.checkSelfPermission(this, Manifest.permission.CAMERA) 
    == PackageManager.PERMISSION_GRANTED) {
    // Permission is granted, proceed with camera access
} else {
    // Permission is not granted, request it
}
```

### 6.3 Requesting Runtime Permissions

#### Modern Approach (ActivityResultLauncher) 

**Single Permission:**
```java
import androidx.activity.result.ActivityResultLauncher;
import androidx.activity.result.contract.ActivityResultContracts;

private ActivityResultLauncher<String> requestPermissionLauncher =
    registerForActivityResult(new ActivityResultContracts.RequestPermission(), isGranted -> {
        if (isGranted) {
            // Permission granted, proceed with the action
        } else {
            // Permission denied, handle accordingly
        }
    });

// Request permission when needed
if (ContextCompat.checkSelfPermission(this, Manifest.permission.CAMERA) 
    != PackageManager.PERMISSION_GRANTED) {
    requestPermissionLauncher.launch(Manifest.permission.CAMERA);
}
```

**Multiple Permissions:**

```java
private ActivityResultLauncher<String[]> requestMultiplePermissionsLauncher =
    registerForActivityResult(new ActivityResultContracts.RequestMultiplePermissions(), 
    permissions -> {
        permissions.forEach((permission, isGranted) -> {
            if (isGranted) {
                // Permission granted
            } else {
                // Permission denied
            }
        });
    });

requestMultiplePermissionsLauncher.launch(new String[]{
    Manifest.permission.CAMERA,
    Manifest.permission.ACCESS_FINE_LOCATION
});
```

#### Legacy Approach (requestPermissions)

**Request:**
```java
import androidx.core.app.ActivityCompat;

ActivityCompat.requestPermissions(this, new String[]{Manifest.permission.CAMERA}, 1);
```

**Handle Result:**
```java
@Override
public void onRequestPermissionsResult(int requestCode, String[] permissions, 
                                     int[] grantResults) {
    super.onRequestPermissionsResult(requestCode, permissions, grantResults);
    if (requestCode == 1) {
        if (grantResults.length > 0 && 
            grantResults[0] == PackageManager.PERMISSION_GRANTED) {
            // Permission granted
        } else {
            // Permission denied
        }
    }
}
```


## 7. Creating Custom Permissions

### 7.1 Defining Custom Permissions

Custom permissions are defined in `AndroidManifest.xml` using the `<permission>` tag:

```xml
<permission
    android:name="com.example.myapp.permission.CUSTOM_PERMISSION"
    android:protectionLevel="dangerous"
    android:label="@string/permission_label"
    android:description="@string/permission_description"
    android:permissionGroup="android.permission-group.STORAGE" />
```

**String Resources (`res/values/strings.xml`):**

```xml
<string name="permission_label">Access Custom Data</string>
<string name="permission_description">Allows the app to access custom app data.</string>
```

or Directly:
```xml
<permission 
    android:label="Allows reading user information" 
    android:name="com.apphacking.privacy.USER_INFO" 
    android:protectionLevel="dangerous"/>
```

### 7.2 Requesting Custom Permissions

Other apps request custom permissions like standard permissions:
```xml
<uses-permission android:name="com.example.myapp.permission.CUSTOM_PERMISSION" />
```

> [!note] If custom permission has `dangerous` protection level, it must be requested at runtime


## 8. Permission Request Process

### Permission Request Flow:

1. Developer declares needed permissions in AndroidManifest.xml
2. At installation/runtime, system processes these requests
3. For dangerous permissions, system prompts user for approval
4. If granted, permission is recorded in `packages.xml`
5. Application can now access the protected functionality

### Technical Implementation:

1. App's UID is added to the group that has the requested permission
2. This grants the Linux-level access needed for the permission
3. System tracks granted state in `packages.xml`



## References

- [Android Permissions Overview](https://developer.android.com/guide/topics/permissions/overview)
- [Requesting Runtime Permissions](https://developer.android.com/training/permissions/requesting)
- [Defining Custom Permissions](https://developer.android.com/guide/topics/permissions/defining)
- [Android Core AndroidManifest.xml](https://android.googlesource.com/platform/frameworks/base.git/+/refs/heads/main/core/res/AndroidManifest.xml)
- [App Permissions Best Practices](https://developer.android.com/training/permissions/usage-notes)
- https://blog.oversecured.com/Common-mistakes-when-using-permissions-in-Android/