# Agora Flutter RTM SDK - AI Coding Agent Instructions

## Project Overview
This is a Flutter plugin wrapping Agora's Real-Time Messaging (RTM) native SDKs for Android and iOS. The plugin uses **method channels** and **event channels** to bridge Dart with native platform code.

## Architecture

### Three-Layer Design
1. **Dart Layer** ([lib/](lib/)): Public API surface with client, channel, and call manager classes
2. **Platform Channel** ([lib/src/agora_rtm_plugin.dart](lib/src/agora_rtm_plugin.dart)): Routing layer that dispatches method calls to native platforms via `caller` parameter (`AgoraRtmClient#static`, `AgoraRtmClient`, `AgoraRtmChannel`, `AgoraRtmCallManager`)
3. **Native Layer**: Android (Kotlin) and iOS (Swift) implementations that wrap Agora native SDKs

### Multi-Client Instance Pattern
- Plugin supports **multiple RTM client instances** indexed by `clientIndex` (see [AgoraRtmPlugin.kt](android/src/main/kotlin/io/agora/agorartm/AgoraRtmPlugin.kt#L27) and [SwiftAgoraRtmPlugin.swift](ios/Classes/SwiftAgoraRtmPlugin.swift#L15))
- Each client maintains its own event channel (`io.agora.rtm.client${clientIndex}`) for native→Dart events
- Channels are sub-resources of clients, with event channels like `io.agora.rtm.client${clientIndex}.channel${channelId}`

### Event Flow Pattern
- **Dart → Native**: Method channel calls with `{"caller": "...", "arguments": {...}}`
- **Native → Dart**: Event channels with `{"event": "eventName", "data": {...}}`
- Native code uses `StreamHandler` to push events; Dart uses `StreamSubscription` to listen

## Key Conventions

### Data Serialization
- **Extensions for JSON mapping**: See [Extensions.kt](android/src/main/kotlin/io/agora/agorartm/Extensions.kt) and [Extensions.swift](ios/Classes/Extensions.swift)
- All RTM model objects (messages, attributes, invitations) have `toJson()` and `fromJson()` methods
- Use `json_serializable` for Dart enums in [utils.dart](lib/src/utils.dart) - run `flutter pub run build_runner build` after adding enums

### Error Handling
- Native methods return `{"errorCode": <int>, "result": <value>}`
- Dart throws `AgoraRtmClientException` or `AgoraRtmChannelException` when `errorCode != 0`
- Always check error codes in native callbacks (e.g., `AgoraRtmErrorCode` on iOS, `ErrorInfo` on Android)

### Resource Management
- Clients track channels in `HashMap<String, RtmChannel>` (Android) or `[String: AgoraRtmChannel]` (iOS)
- Call invitations stored by `hashCode()` in `RTMCallManager` - critical for mapping between Dart and native objects
- Always cleanup: cancel event subscriptions, release channels, destroy clients in reverse order

## Development Workflows

### Adding New RTM Features
1. **Update Dart API**: Add method to [agora_rtm_client.dart](lib/src/agora_rtm_client.dart), [agora_rtm_channel.dart](lib/src/agora_rtm_channel.dart), or [agora_rtm_call_manager.dart](lib/src/agora_rtm_call_manager.dart)
2. **Add Platform Routing**: Update `AgoraRtmPlugin.callMethodFor*` in [agora_rtm_plugin.dart](lib/src/agora_rtm_plugin.dart) if needed
3. **Implement Android**: Add case in `handleClientMethod`/`handleChannelMethod` in [AgoraRtmPlugin.kt](android/src/main/kotlin/io/agora/agorartm/AgoraRtmPlugin.kt)
4. **Implement iOS**: Add case in `handleClientMethod`/`handleChannelMethod` in [SwiftAgoraRtmPlugin.swift](ios/Classes/SwiftAgoraRtmPlugin.swift)
5. **Add Extensions**: Create `toJson()`/`fromJson()` for new data types in both platforms

### Testing Changes
- Run example app: `cd example && flutter run`
- Update [example/lib/main.dart](example/lib/main.dart) with `<YOUR_APPID>` from Agora dashboard
- Test on both Android and iOS simulators/devices

### Code Generation
- After modifying enums in [utils.dart](lib/src/utils.dart): `flutter pub run build_runner build`
- Check [utils.g.dart](lib/src/utils.g.dart) for generated code - don't edit this file manually

## Common Patterns

### Callback Pattern (Android)
```kotlin
object : Callback<ReturnType>(result, handler) {
    override fun toJson(responseInfo: ReturnType): Any {
        return responseInfo.toJson()
    }
}.onSuccess(value) // or use in completion handler
```

### Event Emission (Both Platforms)
```kotlin
// Android
sendEvent("eventName", hashMapOf("key" to value))
```
```swift
// iOS
sendEvent(eventName: "eventName", params: ["key": value])
```

### Client Index Retrieval
All native methods receive `clientIndex` in params - use it to look up the client instance:
```kotlin
val clientIndex = (params?.get("clientIndex") as? Int)?.toLong()
val agoraClient = clients[clientIndex]
```

## Android 16KB Page Size Support

**Critical for Google Play (Nov 1, 2025+ requirement)**: Apps targeting Android 15+ must support 16KB page sizes.

### Plugin Requirements
This plugin depends on `io.agora.rtm:rtm-sdk:1.5.3`, a native library. To ensure 16KB support:

1. **Update Android Gradle Plugin (AGP)**: Requires AGP 8.5.1+ for automatic 16KB zip alignment
   - Current: AGP 7.1.2 in [android/build.gradle](android/build.gradle#L12)
   - Update: `classpath 'com.android.tools.build:gradle:8.5.1'` or higher

2. **Verify Agora SDK Compatibility**: Check if `io.agora.rtm:rtm-sdk:1.5.3` supports 16KB
   - Contact Agora or check their documentation for 16KB-compatible SDK versions
   - If incompatible, update to latest Agora RTM SDK version that supports 16KB

3. **Update compileSdkVersion**: Minimum SDK 35 recommended for 16KB testing
   - Current: `compileSdkVersion 31` in [android/build.gradle](android/build.gradle#L26)
   - Update to: `compileSdkVersion 35` or higher

### For Apps Using This Plugin
Host apps must ensure their entire dependency tree is 16KB-compatible:

```gradle
// In host app's android/build.gradle or build.gradle.kts
android {
    compileSdk = 35
    
    // AGP 8.5.1+ handles this automatically
    // For AGP 8.5 or lower, use:
    packagingOptions {
        jniLibs {
            useLegacyPackaging = true  // Compresses .so files (larger install size)
        }
    }
}
```

### Testing 16KB Compliance
```bash
# Check if plugin's .so files are 16KB aligned
zipalign -c -P 16 -v 4 <your-app>.apk

# Test on 16KB emulator
adb shell getconf PAGE_SIZE  # Should return 16384
```

### Migration Checklist
- [ ] Update AGP to 8.5.1+ in plugin's [android/build.gradle](android/build.gradle)
- [ ] Update compileSdkVersion to 35+ 
- [ ] Verify Agora RTM SDK supports 16KB (update if needed)
- [ ] Test on Android 15+ emulator with 16KB page size
- [ ] Document 16KB requirements in README.md for plugin users

## Critical Files
- [AgoraRtmPlugin.kt](android/src/main/kotlin/io/agora/agorartm/AgoraRtmPlugin.kt) & [SwiftAgoraRtmPlugin.swift](ios/Classes/SwiftAgoraRtmPlugin.swift): Method routing and plugin initialization
- [RTMClient.kt](android/src/main/kotlin/io/agora/agorartm/RTMClient.kt) & [RTMClient.swift](ios/Classes/RTMClient.swift): Client lifecycle and event delegation
- [Extensions.kt](android/src/main/kotlin/io/agora/agorartm/Extensions.kt) & [Extensions.swift](ios/Classes/Extensions.swift): Data transformation layer
- [android/build.gradle](android/build.gradle): Plugin build configuration (AGP version, SDK levels, Agora dependency)
