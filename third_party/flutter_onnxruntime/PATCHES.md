# Vendored dependency: `flutter_onnxruntime`

This is an **unmodified copy of `flutter_onnxruntime` 1.8.3** from pub.dev, except
for one patch that lowers its iOS deployment target from 16.0 to 15.0.

## Why

`flutter_onnxruntime` ships ORT through its own plugin, and upstream hard-codes
`s.platform = :ios, '16.0'` in `ios/flutter_onnxruntime.podspec`
(and `.iOS("16.0")` in `ios/flutter_onnxruntime/Package.swift`). CocoaPods
refuses to resolve any iOS target below the highest pod requirement, so that one
line pinned the *entire* VeneraX app to iOS 16+ — everything else in the
dependency graph only needs iOS 13 or lower.

Nothing in the plugin actually requires iOS 16: the sources are a thin wrapper
over the ONNX Runtime C API and contain no `@available` / `#available` guards.

## What is patched

| File | Change |
| --- | --- |
| `ios/flutter_onnxruntime.podspec` | `:ios, '16.0'` → `:ios, '15.0'` |
| `ios/flutter_onnxruntime/Package.swift` | `.iOS("16.0")` → `.iOS("15.0")` (Swift Package Manager path) |

`macos/`, `android/`, `windows/`, `linux/`, `lib/` and the bundled ONNX Runtime
sources are untouched, so every other platform behaves exactly as before.

## Caveats

* `onnxruntime-objc` 1.23.0 (the pod this plugin depends on) declares
  `"ios": "15.1"` in its own podspec, so `ios/Podfile` keeps `platform :ios, '15.1'`
  purely to satisfy CocoaPods dependency resolution. The **app** itself targets
  iOS 15.0 (`IPHONEOS_DEPLOYMENT_TARGET` in `Runner.xcodeproj`,
  `MinimumOSVersion` in `Flutter/AppFrameworkInfo.plist`), and the Podfile's
  `post_install` hook builds every pod at 15.0, so the resulting IPA installs and
  runs on iOS 15.0.
* The prebuilt ONNX Runtime 1.23.0 framework inside `onnxruntime-c` is itself
  compiled for 15.1, so linking produces a harmless
  `object file ... built for newer iOS version (15.1)` warning. Since it is a
  *static* framework it is folded into the app binary, whose minimum comes from
  the app target, so iOS 15.0 still launches fine.
* If the "built for newer iOS" warning is unacceptable, raise both
  `ios/Podfile`'s `platform` and the app target to `15.1` — everything else stays
  as is.
* Anything below iOS 15.0 would require downgrading ONNX Runtime to ≤ 1.20
  (built for iOS 13.0) *and* replacing the two vendored headers in
  `ios/flutter_onnxruntime/Sources/flutter_onnxruntime_objc/vendor/` with the
  matching 1.20 versions — do not mix ORT versions.

## Updating

1. Download the new package version from pub.dev into a temp dir.
2. Delete everything here except this file, then copy the new contents in.
3. Re-apply the two version patches above (search for `PATCHED (VeneraX)`).
4. Keep `pubspec.yaml` of the app pointing at `third_party/flutter_onnxruntime`.