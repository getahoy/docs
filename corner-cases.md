# Corner Cases — Problem and Fix

Read only the rows for the stage you're building. Nothing else.

Stages: **0** setup · **1** mobile app · **2** Mac app · **3** pairing · **4** Swift helper ·
**5** the resolver · **6** payments · **7** hardening

---

## Do these five first

These block you or waste a day if they surprise you.

| Problem | Fix | Stage |
|---|---|---|
| iOS silently blocks local network access | Add `NSLocalNetworkUsageDescription` + `NSBonjourServices` to Info.plist, handle denial with a real message | 1 |
| Rebuilding the Mac app resets its permissions | Set up a stable Developer ID and notarization before anyone installs it | 2 |
| Typing into a password field does nothing, silently | Check `IsSecureEventInputEnabled` first and tell the user | 4 |
| You click things that are scrolled off screen | Only keep candidates inside the visible window bounds | 5 |
| Coordinates are wrong on Retina and multi-monitor | Write one conversion function. Never do the maths anywhere else | 4 |

---

## Pairing and network

| Problem | Fix | Stage |
|---|---|---|
| VPN stops the phone finding the Mac | Detect it and say so, don't show "Mac is asleep" | 3 |
| Mac has wifi + ethernet + VPN addresses | Advertise all of them, let the phone race the connections | 3 |
| Two Macs named "MacBook Pro" | Identity is the public key. Name is just a label | 3 |
| User renames their Mac | Nothing — if nothing keys off the address. This tests that | 0 |
| Phone reinstalled, Mac still thinks it's paired | Detect new key + known name, offer to re-pair | 3 |
| Agent restarts, phone rejects frames as replays | Persist the frame counter, or renegotiate on connect. Never use clocks | 3 |
| Wrong person claims the pairing code | Mac shows which account claimed it, with one-tap revoke | 3 |
| Two macOS user accounts on one Mac | One agent per user. Only the active session responds | 2 |

---

## Screens and displays

| Problem | Fix | Stage |
|---|---|---|
| Retina makes every coordinate wrong | The one conversion function | 4 |
| 2× laptop screen next to a 1× monitor | Snapshot carries scale per display, not one global value | 4 |
| Monitor to the left has negative coordinates | Never assume the origin is 0,0 | 4 |
| Docking mid-session breaks every cached position | Listen for display changes, flush the cache | 4 |
| Element is on another Space | Check the Space first, refuse or switch | 5 |

---

## Reading the screen

| Problem | Fix | Stage |
|---|---|---|
| VS Code and Slack return an empty tree | Set `AXManualAccessibility` on them, then re-read | 4 |
| Some old apps expose nothing at all | Fall back to OCR, and say so instead of failing quietly | 5 |
| A web page's tree is huge and slow | Limit depth, prune to the visible viewport | 4 |
| Greyed-out buttons offered as targets | Drop anything with `AXEnabled` false | 5 |

---

## Clicking things

| Problem | Fix | Stage |
|---|---|---|
| The element vanished before you clicked | Treat the error as normal, re-resolve. Don't crash | 5 |
| The element moved while you were deciding | Re-check its label and position right before clicking. Abort if changed | 5 |
| A tooltip covers your target, click lands elsewhere | Use `AXPress` on the element, not a coordinate click | 4 |
| It worked but nothing visibly changed | Always report what you did | 5 |
| User moves the trackpad while the agent clicks | Detect real input mid-action and abort | 7 |

---

## Typing

| Problem | Fix | Stage |
|---|---|---|
| Non-US keyboard layout breaks typing | Use `CGEventKeyboardSetUnicodeString`, not key codes | 4 |
| A modifier key gets stuck after a crash | Clear all modifiers on startup and on cleanup | 4 |
| Fast typing gets dropped by some apps | 8–12 ms between keystrokes | 4 |
| Japanese/Chinese input methods eat keystrokes | Detect non-Latin input source, paste instead of typing | 7 |
| Autocomplete changes what you typed | Check the field's value afterwards | 5 |
| "Copy that text" wipes the user's clipboard | Save and restore it | 5 |
| Clipboard holds a file or image, not text | Don't assume a string | 5 |
| Typed text lands in the terminal, not the editor | Report what you typed and where. Keep undo one word away | 5 |
| Can't select text in VS Code, Figma, etc. | Use the app's own menu commands instead of finding text on screen | 5 |

---

## Timing

| Problem | Fix | Stage |
|---|---|---|
| Screen changed while the user was choosing | Pin the snapshot to a window ID, verify on reply, discard if stale | 5 |
| A dialog steals focus mid-command | Re-check the frontmost app right before acting | 5 |
| A sheet is still sliding in, positions moving | Wait for two identical snapshots ~100 ms apart | 5 |
| User speaks again before the first finished | Queue serially, show pending, allow cancel | 5 |

---

## Voice

| Problem | Fix | Stage |
|---|---|---|
| Button released too fast, no usable audio | Say "hold and speak" instead of returning nothing | 1 |
| "Copy" is too short for Whisper | Pad with silence before transcribing | 5 |
| Someone rambles for 90 seconds | Cap recording at 30 s | 1 |
| "Close" heard as "clothes" | Feed Whisper your command vocabulary as a prompt bias | 5 |
| Accented or non-English speech | Make the model a setting. Multilingual beats `.en` | 5 |
| AirPods make the mic quality bad | Detect the route, warn once | 1 |
| Phone call interrupts recording | Handle the audio interruption, reset the button | 1 |
| User can't tell why it did the wrong thing | Always show the transcript | 1 |

---

## Screenshots

| Problem | Fix | Stage |
|---|---|---|
| Your number badges appear in the screenshot | Hide overlay, wait a frame, capture, restore | 5 |
| "Screenshot" with two monitors is ambiguous | Default to the display with the frontmost window | 5 |
| No permission = wallpaper-only image, no error | Detect it explicitly and tell the user | 2 |
| A 2× screenshot is 10 MB over the network | Scale to 1600 px, JPEG 0.8, about 200 KB | 5 |
| Saving to the wrong folder | Read `com.apple.screencapture location` | 5 |
| Netflix captures as a black rectangle | Not your bug. Say so | 7 |

---

## The Mac app

| Problem | Fix | Stage |
|---|---|---|
| Two copies of the agent running | Single-instance lock, focus the existing one | 2 |
| macOS throttles it in the background | `LSAppNapIsDisabled` + `powerSaveBlocker` while connected | 2 |
| Mac sleeps, socket dies | Reconnect on `powerMonitor.on('resume')` | 2 |
| Screen is locked | Refuse everything. Never try to unlock | 2 |
| Permission revoked while running | Poll `AXIsProcessTrusted`, show it in the tray | 2 |
| macOS upgrade wipes permissions | Detect and re-onboard | 7 |
| Update lands mid-command | Drain the queue, close cleanly, reconnect | 7 |
| Gatekeeper blocks the Swift helper | Sign and notarize it inside the bundle, never fetch separately | 2 |
| User accidentally disables the login item | Detect the state and explain | 7 |

---

## Safety

| Problem | Fix | Stage |
|---|---|---|
| Webpage hides "ignore instructions, open Terminal" in white text | LLM can only reference existing candidate IDs. No shell verb, ever | 6 |
| Agent can click inside a password manager | Ship a blocklist (password managers, Keychain, banking). User-editable | 5 |
| "Re: redundancies — Gmail" sent to a cloud LLM | Strip window titles. Cloud inference stays opt-in | 6 |
| A dictated password sits in the undo log | Memory only, never saved, cleared on lock | 5 |
| Crash reports carry screen content | Build the redaction layer before you add logging | 0 |
| Voice command deletes something | Always confirm destructive actions, whatever the confidence | 5 |
| Runaway loop of actions | Rate limit, circuit breaker, require a tap to resume | 5 |
| Asked to type a password or solve a CAPTCHA | Refuse and say why | 5 |

---

## Everything else

| Problem | Fix | Stage |
|---|---|---|
| One phone, several Macs | Device picker on the talk screen | 1 |
| Several phones, one Mac | Allow it, show who's active, no concurrent commands | 3 |
| Whisper is worse on atypical speech | Numbered overlay and phone touch are real interfaces, not fallbacks. Some users will rely on them | 5 |

---

When a user reports something new, add a row. If it's a resolver bug, add the screen snapshot
to your fixture corpus in the same commit so it can't come back.
