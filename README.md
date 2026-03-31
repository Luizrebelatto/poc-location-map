# Poc Location + Map

## Service Overview

### Architecture Summary

A React Native / Expo proof-of-concept mobile application that displays the user's current GPS location on an interactive map. It uses `expo-location` for geolocation and `react-native-maps` (backed by Apple Maps on iOS and Google Maps on Android) for map rendering. The app runs entirely on-device with no backend services, databases, or message queues. Expo Router provides file-based navigation with a single tab screen.

### Component Inventory

| Component           | Type             | Location                                    | Port                         |
| ------------------- | ---------------- | ------------------------------------------- | ---------------------------- |
| Expo Dev Server     | Dev tooling      | `package.json` (scripts.start)              | 8081 (default Metro bundler) |
| React Native Maps   | UI Library       | `app/(tabs)/index.tsx`                      | N/A (native view)            |
| Expo Location       | Native Module    | `app/(tabs)/index.tsx`                      | N/A (OS API)                 |
| Expo Router         | Navigation       | `app/_layout.tsx`, `app/(tabs)/_layout.tsx` | N/A                          |
| Google Maps API Key | External Service | `app.json` (plugins.react-native-maps)      | N/A                          |

### Dependency Map

```
HomeScreen -> expo-location (OS geolocation API) [fallback: no]
HomeScreen -> react-native-maps (Apple Maps iOS / Google Maps Android) [fallback: no]
react-native-maps (Android) -> Google Maps API Key (env var YOUR_GOOGLE_MAPS_API_KEY) [fallback: no]
RootLayout -> expo-router (file-based routing) [fallback: no]
Expo Dev Server -> Metro Bundler (port 8081) [fallback: no]
```

### Startup Order

1. Install dependencies: `npm install`
2. Prebuild native projects: `npx expo prebuild`
3. Install iOS pods: `cd ios && pod install`
4. Start Metro bundler: `npx expo start`
5. Launch on device/simulator: `npx expo run:ios` or `npx expo run:android`

### Shutdown Order

1. Stop the running app on device/simulator
2. Stop Metro bundler (Ctrl+C in terminal)

## Health Checks

| Endpoint                                     | Expected Response         | Checks                    |
| -------------------------------------------- | ------------------------- | ------------------------- |
| Metro Bundler `http://localhost:8081/status` | `packager-status:running` | Metro dev server is alive |

Manual health check commands:

```sh
curl -s http://localhost:8081/status
```

## Failure Runbooks

### Location Permission Denied

**Severity:** Critical
**Blast Radius:** App shows error screen — map is never rendered, core functionality completely unavailable
**Affected Component:** expo-location (`app/(tabs)/index.tsx:19`)
**Fallback Exists:** No (shows static error text "Permissão de localização negada")

**Symptoms:**

- Red error text displayed: "Permissão de localização negada"
- Map never loads

**Log Breadcrumbs:**

- Check `app/(tabs)/index.tsx:20-21` — permission status check sets error state when `status !== "granted"`

**Diagnosis Steps:**

1. On iOS simulator: check Settings > Privacy & Security > Location Services > poclocationmap
2. On Android emulator: check Settings > Apps > poclocationmap > Permissions > Location
3. Verify `Info.plist` contains `NSLocationWhenInUseUsageDescription` key:

```sh
grep -c "NSLocationWhenInUseUsageDescription" ios/poclocationmap/Info.plist
```

4. Verify `app.json` contains Android location permissions:

```sh
grep "ACCESS_FINE_LOCATION" app.json
```

**Resolution Steps:**

1. On device: go to Settings, grant location permission to the app
2. Kill and restart the app (permission is only requested once via `requestForegroundPermissionsAsync`)
3. If permission dialog never appears on iOS, verify the plist keys exist in `ios/poclocationmap/Info.plist` (lines 48-53)

**Rollback Procedure:**

1. N/A — this is a runtime permission issue, not a code deployment

**Prevention:**

- Test on physical devices where location services can be toggled
- Consider adding a "retry" button that re-requests permission instead of showing a dead-end error

---

### Location Acquisition Failure / Timeout

**Severity:** High
**Blast Radius:** App stuck on loading spinner indefinitely — user sees "Obtendo localização..." forever
**Affected Component:** expo-location (`app/(tabs)/index.tsx:25`)
**Fallback Exists:** No (no timeout, no error handling on `getCurrentPositionAsync`)

**Symptoms:**

- Infinite loading spinner with text "Obtendo localização..."
- No error message displayed

**Log Breadcrumbs:**

- Check `app/(tabs)/index.tsx:25` — `Location.getCurrentPositionAsync({})` is called with no timeout option
- The async IIFE at line 18 has no `.catch()` — unhandled promise rejection if the call throws

**Diagnosis Steps:**

1. Check if Location Services are enabled at OS level (iOS: Settings > Privacy > Location Services toggle)
2. On iOS simulator: Features > Location > set to a specific location (simulators have no real GPS)
3. Check Metro bundler logs for unhandled promise rejection warnings

**Resolution Steps:**

1. Ensure Location Services are enabled at the OS level
2. On simulator: set a simulated location via Xcode (Debug > Simulate Location) or Simulator menu (Features > Location)
3. If on a physical device indoors, move to a location with GPS signal or connect to Wi-Fi for network-based location

**Rollback Procedure:**

1. Force-quit and relaunch the app

**Prevention:**

- Add a timeout to `getCurrentPositionAsync` call: `{ timeout: 10000 }`
- Add a `.catch()` handler to the async IIFE to display errors to the user
- Consider using `getLastKnownPositionAsync()` as a fast fallback while waiting for fresh GPS fix

---

### Google Maps API Key Missing or Invalid (Android)

**Severity:** Critical
**Blast Radius:** Map view fails to render on Android — blank screen or crash
**Affected Component:** react-native-maps, Google Maps SDK (`app.json:57-58`)
**Fallback Exists:** No

**Symptoms:**

- Blank/grey map area on Android
- Possible crash on Android with "API key not found" in logcat
- iOS unaffected (uses Apple Maps by default)

**Log Breadcrumbs:**

- Check `app.json:57` — `iosGoogleMapsApiKey` is set to the literal string `"process.env.YOUR_GOOGLE_MAPS_API_KEY"` (not an actual env var interpolation)
- Check `app.json:58` — same issue for `androidGoogleMapsApiKey`
- The `.env` file exists at project root but appears empty

**Diagnosis Steps:**

1. Verify the API key configuration in `app.json`:

```sh
grep -A2 "GoogleMapsApiKey" app.json
```

2. Check if `.env` file has the key set:

```sh
cat .env
```

3. On Android, check logcat for Google Maps errors:

```sh
adb logcat | grep -i "maps\|api.key\|google"
```

**Resolution Steps:**

1. Obtain a valid Google Maps API key from Google Cloud Console with "Maps SDK for Android" and "Maps SDK for iOS" enabled
2. Replace the literal string in `app.json` with the actual API key, or configure proper env var interpolation using `expo-constants` or `app.config.js`
3. Run `npx expo prebuild --clean` to regenerate native projects with the new key
4. Rebuild: `npx expo run:android`

**Rollback Procedure:**

1. Revert `app.json` changes and rebuild

**Prevention:**

- Convert `app.json` to `app.config.js` or `app.config.ts` to enable proper `process.env` interpolation
- Document required env vars in a `.env.example` file
- Validate API key presence at build time

---

### Metro Bundler Port Conflict

**Severity:** Medium
**Blast Radius:** Development server fails to start — cannot load JS bundle on device/simulator
**Affected Component:** Expo Dev Server (`package.json:6`, default port 8081)
**Fallback Exists:** Yes (Metro can use alternate port via `--port` flag)

**Symptoms:**

- "Something is already running on port 8081" error when running `npx expo start`
- App on device shows "Unable to load script" red screen

**Log Breadcrumbs:**

- Terminal output from `npx expo start` will show the port conflict message

**Diagnosis Steps:**

1. Check what is using port 8081:

```sh
lsof -i :8081
```

**Resolution Steps:**

1. Kill the conflicting process:

```sh
kill $(lsof -t -i :8081)
```

2. Or start Metro on an alternate port:

```sh
npx expo start --port 8082
```

**Rollback Procedure:**

1. N/A — development tooling only

**Prevention:**

- Ensure no other Metro instances or React Native projects are running before starting

---

### iOS Build Failure — CocoaPods Out of Sync

**Severity:** Medium
**Blast Radius:** Cannot build or run the iOS app from Xcode or CLI
**Affected Component:** iOS native build (`ios/` directory, CocoaPods)
**Fallback Exists:** No

**Symptoms:**

- Xcode build fails with "module not found" errors for native modules (e.g., `ExpoLocation`, `react-native-maps`)
- `pod install` produces version conflicts
- Build error in `.expo/xcodebuild-error.log`

**Log Breadcrumbs:**

- Check `.expo/xcodebuild-error.log` for build errors
- Check `.expo/xcodebuild.log` for full build output

**Diagnosis Steps:**

1. Check for pod install issues:

```sh
cd ios && pod install --repo-update 2>&1 | tail -20
```

2. Verify Podfile.lock is in sync:

```sh
cd ios && pod outdated
```

**Resolution Steps:**

1. Clean and reinstall pods:

```sh
cd ios && rm -rf Pods Podfile.lock && pod install --repo-update
```

2. If that fails, clean the full prebuild:

```sh
npx expo prebuild --clean
cd ios && pod install
```

3. Clean Xcode derived data:

```sh
rm -rf ~/Library/Developer/Xcode/DerivedData/poclocationmap-*
```

4. Rebuild:

```sh
npx expo run:ios
```

**Rollback Procedure:**

1. Restore `ios/Podfile.lock` from git: `git checkout ios/Podfile.lock`
2. Run `cd ios && pod install`

**Prevention:**

- Run `pod install` after every `npm install` that adds/removes native modules
- Commit `Podfile.lock` to version control

---

## Configuration Drift Checks

- **API Key Config**: `app.json:57-58` sets Google Maps API keys to the literal string `"process.env.YOUR_GOOGLE_MAPS_API_KEY"` — this is NOT env var interpolation in a JSON file. The app will receive the literal string as the API key. Convert to `app.config.js`/`app.config.ts` to use `process.env` properly.
- **Empty .env**: The `.env` file at project root exists but appears empty. No API keys are actually configured.

## Appendix

### Detected Endpoints

No HTTP endpoints — this is a client-only mobile application.

### Detected Dependencies

| Dependency        | Type             | Connection Details                                                                                       |
| ----------------- | ---------------- | -------------------------------------------------------------------------------------------------------- |
| expo-location     | Native OS API    | `Location.requestForegroundPermissionsAsync()`, `Location.getCurrentPositionAsync()` — no network config |
| react-native-maps | Native Map SDK   | Apple Maps (iOS, no key needed), Google Maps (Android, requires API key via `app.json`)                  |
| Google Maps API   | External Service | API key configured in `app.json` plugins section (currently misconfigured — literal string, not env var) |

### Detected Configuration Files

| File                            | Purpose                                                                  |
| ------------------------------- | ------------------------------------------------------------------------ |
| `app.json`                      | Expo app configuration: name, plugins, permissions, icons, splash screen |
| `tsconfig.json`                 | TypeScript configuration with path aliases (`@/*`)                       |
| `package.json`                  | Dependencies and scripts                                                 |
| `ios/poclocationmap/Info.plist` | iOS permissions (location), URL schemes, device capabilities             |
| `constants/theme.ts`            | App color theme and font definitions                                     |
| `.env`                          | Environment variables (currently empty)                                  |

### Detected Resilience Patterns

None detected. The app has no circuit breakers, retries, rate limiters, or feature flags. The location permission check at `app/(tabs)/index.tsx:19-23` is the only error-handling path, and `getCurrentPositionAsync` at line 25 has no timeout or catch handler.
