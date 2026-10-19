# Setup Guide: Expo and Swift

Hands-on. Run the commands, understand what each one did. By the end you'll have a phone app
on a simulator and a Swift program that reads your Mac's screen.

Do the Swift part first. It takes twenty minutes and proves the riskiest assumption in the
whole project.

---

# Part A — Swift helper

## A1. What you're installing

Swift comes with Xcode. You need the full Xcode, not just the command line tools, because
you'll want Accessibility Inspector later.

```bash
# Install Xcode from the App Store first, then:
sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
sudo xcodebuild -license accept

# Verify
swift --version
```

You should see Swift 5.9 or newer. If `swift --version` points at
`/Library/Developer/CommandLineTools`, the `xcode-select` line above didn't take.

## A2. Create the package

This is a **command-line executable**, not an app. No Xcode project, no storyboards. Swift
Package Manager handles it.

```bash
mkdir -p ~/code/ax-test && cd ~/code/ax-test
swift package init --type executable --name axtest
```

You now have:

```
ax-test/
├── Package.swift          the manifest
└── Sources/
    └── axtest/
        └── main.swift     your code
```

## A3. Set the platform

Open `Package.swift` and replace it with this. The `platforms:` line is what gives you access
to the macOS frameworks.

```swift
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "axtest",
    platforms: [.macOS(.v13)],
    targets: [
        .executableTarget(name: "axtest")
    ]
)
```

## A4. Your first real program

Replace `Sources/axtest/main.swift` with this. It checks Accessibility permission, finds the
frontmost app, and prints its element tree.

```swift
import Foundation
import ApplicationServices
import AppKit

let trusted = AXIsProcessTrustedWithOptions(
    [kAXTrustedCheckOptionPrompt.takeUnretainedValue(): true] as CFDictionary
)

guard trusted else {
    print("Grant Accessibility permission in System Settings, then run again.")
    exit(1)
}

func stringAttribute(_ element: AXUIElement, _ name: String) -> String? {
    var value: CFTypeRef?
    guard AXUIElementCopyAttributeValue(element, name as CFString, &value) == .success,
          let text = value as? String,
          !text.isEmpty
    else { return nil }
    return text
}

func label(of element: AXUIElement) -> String? {
    stringAttribute(element, kAXTitleAttribute as String)
        ?? stringAttribute(element, kAXDescriptionAttribute as String)
        ?? stringAttribute(element, kAXValueAttribute as String)
}

func children(of element: AXUIElement) -> [AXUIElement] {
    var value: CFTypeRef?
    guard AXUIElementCopyAttributeValue(element, kAXChildrenAttribute as CFString, &value) == .success
    else { return [] }
    return value as? [AXUIElement] ?? []
}

func walk(_ element: AXUIElement, depth: Int) {
    guard depth <= 6 else { return }
    let role = stringAttribute(element, kAXRoleAttribute as String) ?? "?"
    if let text = label(of: element) {
        print(String(repeating: "  ", count: depth) + "\(role): \(text)")
    }
    for child in children(of: element) {
        walk(child, depth: depth + 1)
    }
}

guard let app = NSWorkspace.shared.frontmostApplication else { exit(1) }
print("Frontmost: \(app.localizedName ?? "?")\n")

let axApp = AXUIElementCreateApplication(app.processIdentifier)
for window in children(of: axApp) {
    walk(window, depth: 0)
}
```

## A5. Run it

```bash
swift build
.build/debug/axtest
```

**First run will fail and that's expected.** macOS will prompt for Accessibility permission.

**Here's the gotcha that confuses everyone:** the permission attaches to whatever *launched*
the binary, which is Terminal, not `axtest`. So in System Settings → Privacy & Security →
Accessibility, you grant **Terminal**, not your program.

Grant it, quit Terminal completely (Cmd+Q, not just closing the window), reopen, and run
again.

## A6. Try it on things

```bash
# Click on Finder, then within 3 seconds:
sleep 3 && .build/debug/axtest

# Now try VS Code
sleep 3 && .build/debug/axtest
```

**Finder gives you a rich tree.** Buttons, toolbar items, file names.

**VS Code gives you almost nothing.** That is the Electron problem from the docs, and you are
now looking at it directly. This is the single most important thing to see with your own eyes
before you build anything.

To fix it, add this before the `walk` loop:

```swift
AXUIElementSetAttributeValue(
    axApp, "AXManualAccessibility" as CFString, kCFBooleanTrue
)
Thread.sleep(forTimeInterval: 0.4)
```

Rebuild, run against VS Code again. The tree appears. **That 0.4 second sleep is why** the
docs say to wait after setting the attribute.

## A7. Speaking JSON over stdin

This is the shape the real helper takes. Node writes a line, Swift replies with a line.

```swift
struct Request: Decodable {
    let id: String
    let cmd: String
}

struct Response: Encodable {
    let id: String
    let ok: Bool
    let result: String?
}

let encoder = JSONEncoder()

while let line = readLine(strippingNewline: true) {
    guard let data = line.data(using: .utf8),
          let request = try? JSONDecoder().decode(Request.self, from: data)
    else { continue }

    let response = Response(id: request.id, ok: true, result: "got \(request.cmd)")
    guard let out = try? encoder.encode(response),
          let text = String(data: out, encoding: .utf8)
    else { continue }

    print(text)
    fflush(stdout)
}
```

**`fflush(stdout)` is mandatory.** Without it the output sits in a buffer and Node waits
forever for a reply that was already written. This costs people an afternoon.

Test it by hand:

```bash
echo '{"id":"1","cmd":"snapshot"}' | .build/debug/axtest
```

## A8. Calling it from Node

```ts
import { spawn } from 'node:child_process'
import readline from 'node:readline'

const helper = spawn('.build/debug/axtest')
const lines = readline.createInterface({ input: helper.stdout })

lines.on('line', (line) => {
  console.log('reply:', JSON.parse(line))
})

helper.stdin.write(JSON.stringify({ id: '1', cmd: 'snapshot' }) + '\n')
```

Spawn once at startup and keep it alive. Spawning per command costs 60ms you don't have.

## A9. Build for release

```bash
swift build -c release
.build/release/axtest
```

Release is meaningfully faster for tree traversal. Use it for any timing you care about.

---

# Part B — Expo

## B1. Prerequisites

```bash
node -v      # need 20 or newer
```

For iOS you need Xcode (already installed from Part A).

For Android, install Android Studio, then during setup let it install the SDK. Add to your
`~/.zshrc`:

```bash
export ANDROID_HOME=$HOME/Library/Android/sdk
export PATH=$PATH:$ANDROID_HOME/emulator:$PATH:$ANDROID_HOME/platform-tools
```

Then `source ~/.zshrc`.

## B2. The one concept that matters: Expo Go vs development build

This confuses nearly everyone, so get it straight now.

**Expo Go** is an app you download from the App Store. It already contains Expo's standard
native modules. Your JavaScript runs inside it. Fast to start, but you are limited to the
native code Expo Go already has.

**A development build** is *your own app*, compiled with *your own* native modules, with the
dev tooling included. You install it on the simulator or your phone once, then JavaScript
reloads into it just like Expo Go.

**You need a development build**, because the project uses mDNS discovery, libsodium and raw
audio capture. None of those exist in Expo Go.

## B3. Create the project

```bash
cd ~/code
npx create-expo-app@latest mobile
cd mobile
```

Run it the easy way first, just to see it work:

```bash
npx expo start
```

Press `i` for the iOS simulator. You should get a starter screen. This is still Expo Go
territory. Now we leave it.

## B4. Switch to a development build

```bash
npx expo install expo-dev-client
```

Now generate the native projects:

```bash
npx expo prebuild
```

You now have `ios/` and `android/` folders. **Do not hand-edit them.** They are generated from
`app.json`, and `prebuild` will overwrite your changes. Everything you need to change in them
is done through config, which is the next step.

Build and install:

```bash
npx expo run:ios
```

First build takes five to ten minutes. Later ones are much faster. When it finishes you have
your own app on the simulator.

Android:

```bash
npx expo run:android
```

## B5. Config plugins: changing native files without touching them

This is how you add the permissions the project needs. Edit `app.json`:

```json
{
  "expo": {
    "name": "Ahoy",
    "slug": "ahoy",
    "ios": {
      "bundleIdentifier": "com.yourname.ahoy",
      "infoPlist": {
        "NSLocalNetworkUsageDescription": "Ahoy connects to your Mac on your local network.",
        "NSBonjourServices": ["_ahoy._tcp"],
        "NSMicrophoneUsageDescription": "Ahoy uses your microphone for voice commands.",
        "NSCameraUsageDescription": "Ahoy uses your camera to scan the pairing code."
      }
    },
    "android": {
      "package": "com.yourname.ahoy",
      "permissions": [
        "RECORD_AUDIO",
        "CAMERA",
        "CHANGE_WIFI_MULTICAST_STATE",
        "ACCESS_NETWORK_STATE",
        "POST_NOTIFICATIONS"
      ]
    }
  }
}
```

Apply it:

```bash
npx expo prebuild --clean
npx expo run:ios
```

`--clean` regenerates the native folders from scratch. Use it whenever you change config.

Those two iOS keys are the ones from the docs that fail silently if missing. You just added
them.

## B6. Android cleartext

Android blocks `ws://` by default, which is the LAN socket. Create
`plugins/withCleartext.js`:

```js
const { withAndroidManifest } = require('@expo/config-plugins')

module.exports = function withCleartext(config) {
  return withAndroidManifest(config, (cfg) => {
    cfg.modResults.manifest.application[0].$['android:usesCleartextTraffic'] = 'true'
    return cfg
  })
}
```

Register it in `app.json`:

```json
"plugins": ["./plugins/withCleartext.js"]
```

This is the blunt version to get you unblocked. Before you ship, replace it with a network
security config scoped to private ranges only, as the docs describe. Google Play flags the
global flag.

## B7. Daily workflow

```bash
npx expo start --dev-client     # start the JS bundler
# press i for iOS, a for Android
```

You only rebuild (`expo run:ios`) when you add a native module or change config. JavaScript
changes reload instantly.

```bash
npx expo install <package>      # use this, not npm install, for native packages
```

`expo install` picks the version compatible with your Expo SDK. Plain `npm install` will give
you a version that breaks the build.

## B8. Getting it onto your real phone

The simulator can't reach your LAN properly and has no real microphone, so you'll need a
device early in this project.

```bash
npm install -g eas-cli
eas login
eas build:configure
eas build --profile development --platform ios
```

This builds in Expo's cloud and gives you a QR to install from. It needs an Apple Developer
account ($99/year) for a real device.

To build locally instead, plug your phone in, open `ios/ahoy.xcworkspace` in Xcode,
select your device, and press run. You'll need to set a signing team in Xcode first.

---

# Part C — Do this today

Three things, in order. Maybe two hours total.

**1. Run the Swift program against Finder.** See a real accessibility tree.

**2. Run it against VS Code.** See nothing. Then add `AXManualAccessibility` and see it
appear. This is the single most important thing to witness before you build anything, because
the entire resolver rests on it.

**3. Create the Expo project and get a dev build on the simulator.** Just the starter screen.
You're proving the toolchain works, nothing more.

If all three work, every risky assumption in the project is confirmed and you can start
feature 1 with confidence.
