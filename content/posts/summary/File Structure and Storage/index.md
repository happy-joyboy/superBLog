+++
date = '2025-07-30T00:55:36+03:00'
draft = false
title = 'Android System File Structure and Storage'
tags = ['Summary', 'Andorid']
+++

## Storage summary
### **Directories you can see (Non-Root)** (Attack Surface)

#### Public External Storage

- **Path**: `/storage/emulated/0/` (symlinked as `/sdcard/`)
- **Access Level**: World-readable/writable with proper permissions
- **Security Implications**:
    - ANY app can read files here with `READ_EXTERNAL_STORAGE` permission
    - Data persistence even after app uninstallation
    - Potential data leakage vector

##### Key Subdirectories:
```
/sdcard/
├── DCIM/           # Camera images
├── Downloads/      # Downloaded files
├── Pictures/       # User images
├── Music/          # Audio files
└── Android/data/<package_name>/  # App-specific external data
```

#### App-Specific External Storage

- **Path**: `/sdcard/Android/data/<package_name>/`
- **Access**:
    - App can access without `WRITE_EXTERNAL_STORAGE` permission (Android 4.4+)
    - Other apps can access with proper permissions
    - **Scoped Storage** restrictions apply (Android 11+)

#### World-Writable Directory

- **Path**: `/data/local/tmp/`
- **Security Risk**: Any app can write/read files here
- **Common Attack Vector**: Privilege escalation, data exfiltration

### Protected Directories (Root/System Access Required)

- **`/data/data/`**
    - Contains **every installed application's private data**
    - Each app subdirectory owned by unique Linux UID (different users separate directories with their own apps)

**Path**: `/data/data/<package_name>/`
- **Security Model**: App-sandboxed, unique Linux UID per app (different users separate directories with their own apps)
- Contains **every installed application's private data**
- **Contains**:
#### Key App Data Subdirectories:
```
/data/data/<package-name>/
├── databases/          # SQLite databases (populated on first run)
├── shared_prefs/       # SharedPreferences XML files
├── files/             # Private app files
└── cache/             # App cache data
```

#### APK and Binary Locations

- **`/data/app/`**
    - User-installed APK files (decrypted)
    - **Researchers**: APK extraction, reverse engineering source
- **`/data/app-asec/`**
    - Encrypted ASEC containers for "forward locked" apps
    - **Security Focus**: Encryption analysis, DRM bypass research
- **`/data/app-lib/`**
    - Extracted native libraries (.so files)
    - **Researchers**: Native code analysis, ROP/JOP gadgets

#### System App Directories & Configuration

- **`/system/app/`** - Pre-installed system apps
- **`/system/priv-app/`** - Privileged apps with `signatureOrSystem` permissions
- **`/system/vendor/app/`** - Vendor-specific applications
- **`/system/bin/`** - System binaries

##### Package Management
- **`/data/system/packages.xml`**
    - **Critical for researchers**: Package database, UIDs, permissions, signing certificates
    - Maps package names to user IDs and permissions
- **`/data/system/packages.list`**
    - App UIDs, package names, debuggable flags, data paths
    - **Security use**: Permission enumeration, attack surface mapping

##### Certificate Stores
- **`/etc/security/cacerts/`** - System certificate store (root only)
- **`/data/misc/user/0/cacerts-added/`** - User-added certificates
    - **Security researchers**: Certificate pinning bypass, MITM analysis

##### Network Configuration
- **`/etc/apns-conf.xml`** - APN configurations
- **`/data/misc/wifi/`** - WiFi configuration files
    - **Security focus**: Stored network credentials, PSK analysis

##### Multi-User environment 
- **`/data/user/`** - Multi-user data directories
- **`/data/user/0/`** - Device owner data (symlink to `/data/data/`)
- **`/data/system/users/<user ID>/`**
    - User metadata, accounts database (`accounts.db`)
    - Lock screen credentials (`gesture.key`, `password.key`)


### API Access Methods

```java
// Internal storage
Context.getFilesDir()           // /data/data/<pkg>/files/
Context.getDatabasePath()       // /data/data/<pkg>/databases/

// External storage
getExternalFilesDir()          // /sdcard/Android/data/<pkg>/files/
Environment.getExternalStoragePublicDirectory()  // /sdcard/
```

###  **Version-Specific Changes**

#### Android 11+ (API 30)

- **Scoped Storage** mandatory
- Restricted access to `/sdcard/Android/data/`
- Enhanced privacy controls

#### Android 10 (API 29)

- Scoped Storage introduction
- External storage filtering

#### Legacy Versions

- Broader external storage access
- Fewer privacy restrictions


## APK Structure summary

### 📁 Core Files & Directories

| Component             | Description                                                                           | Importance     | Notes                                            |
| --------------------- | ------------------------------------------------------------------------------------- | -------------- | ------------------------------------------------ |
| `AndroidManifest.xml` | Core configuration file with package name, permissions, components, debuggable status | ⭐⭐⭐ Very High  | Binary XML format - requires `apktool` to decode |
| `classes.dex`         | Main Java source code compiled to Dalvik Executable format                            | ⭐⭐⭐ Very High  | Contains primary app logic                       |
| `assets/`             | Custom developer resources (certs, configs, etc.)                                     | ⭐⭐ High        | Often contains security-relevant data            |
| `lib/`                | Native C/C++ shared object (.so) libraries                                            | ⭐⭐ High        | Architecture-specific folders; harder to reverse |
| `resources.arsc`      | Compiled resources (strings, colors, UI attributes)                                   | ⭐ Low-Moderate | Precompiled resources linking code to assets     |
| `res/`                | Images, UI resources, language strings                                                | ⭐ Moderate     | Predefined resource types                        |
| `META-INF/`           | App signing information and verification data                                         | ⭐ Moderate     | Contains signature files and hashes              |
| `com/`                | XML fragments and general files                                                       | ❌ Low          | Usually not useful for reverse engineering       |

### META-INF Directory Contents

- **`MANIFEST.MF`**: File names/hashes (SHA256 Base64) for all APK files
- **`CERT.SF`**: Names/hashes of corresponding `MANIFEST.MF` lines
- **`CERT.RSA`**: Public key and signature of `CERT.SF`

### lib Directory Structure

- Contains architecture-specific subdirectories:
    - `armeabi-v7a/`
    - `x86/`
    - `arm64-v8a/`
