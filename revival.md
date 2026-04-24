Excellent question! You're right that API keys should NEVER be committed to GitHub. Let me show you exactly where and how you'd inject API keys when reviving this project.

## Where API Keys are EXCLUDED (from .gitignore)

Looking at the project's `.gitignore`, I can see they intentionally exclude:
- **`android/key.properties`** - Android signing keys
- **`android/*.jks`** and **`android/*.keystore`** - Android keystores
- **`android/local.properties`** - Local Android SDK configuration
- All **`*.lock` files** (except `pubspec.lock`) which could contain secrets

This is the RIGHT approach - secrets are never versioned.

## How to Inject API Keys When You Revive the Project

### **1. For Flutter/Dart Code (Runtime API Calls)**

Create these files locally (they're already in `.gitignore`):

**File: `lib/config/api_keys.dart`** (Create this - it's NOT in the repo)
```dart
// This file is local-only and NOT committed to Git
class ApiKeys {
  static const String tidalApiKey = 'YOUR_TIDAL_KEY_HERE';
  static const String qobuzApiKey = 'YOUR_QOBUZ_KEY_HERE';
  static const String deezerApiKey = 'YOUR_DEEZER_KEY_HERE';
  // ... other keys
}
```

Then import and use it in your code:
```dart
import 'lib/config/api_keys.dart';

// Use the key
final headers = {'Authorization': 'Bearer ${ApiKeys.tidalApiKey}'};
```

**Better approach - Environment Variables:**
```dart
// Use the `flutter_dotenv` package (add to pubspec.yaml)
final tidalKey = dotenv.env['TIDAL_API_KEY'] ?? '';
```

Create **`.env`** file locally (add `.env` to `.gitignore`):
```
TIDAL_API_KEY=your_key_here
QOBUZ_API_KEY=your_key_here
DEEZER_API_KEY=your_key_here
```

### **2. For Android Build Configuration**

**File: `android/key.properties`** (Local only, already in .gitignore)
```properties
storeFile=path/to/your/keystore.jks
storePassword=your_keystore_password
keyAlias=your_key_alias
keyPassword=your_key_password
```

The `android/app/build.gradle.kts` will reference this file:
```kotlin
val keyProperties = Properties()
val keyPropertiesFile = rootProject.file("key.properties")
if (keyPropertiesFile.exists()) {
    keyProperties.load(FileInputStream(keyPropertiesFile))
}

android {
    signingConfigs {
        create("release") {
            storeFile = file(keyProperties["storeFile"].toString())
            storePassword = keyProperties["storePassword"].toString()
            keyAlias = keyProperties["keyAlias"].toString()
            keyPassword = keyProperties["keyPassword"].toString()
        }
    }
}
```

### **3. For iOS Build Configuration**

**File: `ios/Certificates/`** (Local only, already in .gitignore via `.symlinks/`)
- Apple provisioning profiles
- Code signing certificates
- Distribution certificates

These are handled via Xcode settings or environment variables in GitHub Actions.

### **4. For CI/CD (GitHub Actions)**

When you revive the project, set GitHub **Secrets** in your repository settings:

**GitHub Settings → Secrets and variables → Actions**

Add these secrets:
- `TIDAL_API_KEY`
- `QOBUZ_API_KEY`
- `DEEZER_API_KEY`
- `ANDROID_KEYSTORE_BASE64` (base64 encoded keystore)
- `ANDROID_KEYSTORE_PASSWORD`
- `ANDROID_KEY_ALIAS`
- `ANDROID_KEY_PASSWORD`
- `APPLE_SIGNING_CERTIFICATE_P12_BASE64`
- `APPLE_SIGNING_CERTIFICATE_PASSWORD`
- `APPLE_PROVISIONING_PROFILE_BASE64`

Then in `.github/workflows/release.yml`, use them like:
```yaml
- name: Build Release
  env:
    TIDAL_API_KEY: ${{ secrets.TIDAL_API_KEY }}
    QOBUZ_API_KEY: ${{ secrets.QOBUZ_API_KEY }}
    ANDROID_KEYSTORE_BASE64: ${{ secrets.ANDROID_KEYSTORE_BASE64 }}
  run: |
    # Decode and use the keystore
    echo $ANDROID_KEYSTORE_BASE64 | base64 -d > android/app/release.jks
    # Build the app using environment variables
```

## Step-by-Step Process for Revival

**When you fork and want to rebuild:**

1. **Clone your fork:**
   ```bash
   git clone https://github.com/YOUR_USERNAME/SpotiFLAC-Mobile.git
   cd SpotiFLAC-Mobile
   ```

2. **Create local config files (NOT committed):**
   ```bash
   # Create .env for Flutter
   echo "TIDAL_API_KEY=your_key" > .env
   echo "QOBUZ_API_KEY=your_key" >> .env
   
   # Create key.properties for Android signing
   echo "storeFile=/path/to/keystore.jks" > android/key.properties
   echo "storePassword=password" >> android/key.properties
   ```

3. **Get your API keys:**
   - Tidal API: Register at [Tidal Developer](https://developer.tidal.com/)
   - Qobuz API: Register at [Qobuz Developers](https://developer.qobuz.com/)
   - Deezer API: Register at [Deezer Developers](https://developers.deezer.com/)

4. **Get signing certificates:**
   - Create Android keystore: `keytool -genkey -v -keystore app-release.jks -keyalg RSA -keysize 2048 -validity 10000`
   - Get iOS certificates from Apple Developer account

5. **For CI/CD, set secrets** in GitHub repo settings (as mentioned above)

## The Safety Pattern in SpotiFLAC

Notice the `.gitignore` already has patterns for:
- **`android/local.properties`** - Local SDK paths (not committed)
- **`*.jks, *.keystore`** - Signing keys (not committed)
- **`key.properties`** - Build properties with passwords (not committed)

This means the original developers already follow best practices! You just need to add the same files locally when you want to build.

## Summary

- **Pushed code:** Has NO API keys ✅
- **Local development:** Create `lib/config/api_keys.dart` or `.env` with your keys (Git-ignored)
- **For Android signing:** Create `android/key.properties` locally (Git-ignored)
- **For CI/CD:** Use GitHub Secrets, never commit keys
- **Document it:** Create `SETUP.md` explaining how contributors should set up API keys locally

This way your fork stays safe while you have full ability to build and test the app!
