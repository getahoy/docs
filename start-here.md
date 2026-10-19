# Start Here

The whole project. Read this one. The others are for detail.

---

## What it is

Talk to your Mac from your phone. You say "click the plus button" and it clicks it.

It only does what it can see on screen right now. No planning, no wandering off.
A remote control, not an assistant.

---

## Three pieces

| Piece | Job | Built with |
|---|---|---|
| Phone | Hears you, shows you things | React Native |
| Mac app | Thinks. Turns words into a decision | Electron + Node |
| Swift program | Does the clicking and typing | Swift, small |

JavaScript can't touch macOS internals. So one small native piece does the touching, and
everything smart stays in TypeScript where it's easy to test and change.

The Swift piece decides nothing. It answers "what's on screen?" and obeys "click this."

---

## How they connect

**Phone to Mac** — directly over your wifi. Scan a QR code on the Mac once, then they know
each other. The internet is not involved.

**Mac app to Swift program** — same computer, so they pass text messages through a pipe.

Same wifi only for now. A server in the middle can be added later for when the Mac is far
away. The code is built so that's an addition, not a rewrite.

---

## What happens when you speak

1. You hold the button and talk.
2. The Mac writes down what you said.
3. It asks the Swift program what's on screen.
4. It matches your words to that list.
5. It clicks the match and tells your phone what it did.

If two things match, your phone shows pictures and you tap the right one.

**It asks instead of guessing.** That rule is the whole product.

---

## Build plan

One feature at a time, each finished completely. Stop and commit after each.

| # | Feature | Done when |
|---|---|---|
| 1 | Setup | Repos, shared message types, CI green |
| 2 | Auth | Sign up and log in works, sessions persist |
| 3 | Swift helper | It can see the screen and click things |
| 4 | Mac agent shell | Installs, menubar, talks to the helper |
| 5 | Pairing | Scan QR, phone and Mac connected |
| 6 | Talk + speech | Hold, speak, the Mac shows what you said |
| 7 | Matching | You're talking to your Mac. **The product works here** |
| 8 | Devices screen | Real paired devices |
| 9 | Dashboard + profile | Real command history |
| 10 | Menu commands | Any app's menu items by voice |
| 11 | Screenshots | Capture, OCR, send to phone |
| 12 | LLM tier | The fallback resolver |
| 13 | Website + payments | People can buy it |
| 14 | Hardening | The corner cases |

UI stays visually plain until the Figma designs exist. Logic, state and components are built
to full quality from the start, with every visual value read from a tokens file so applying
the design later is one file, not forty.

## Six rules that matter

1. **Write the message formats first.** Before either app. Then a change breaks both sides at
   once and nothing drifts.
2. **Encrypt on wifi anyway.** Skipping it now means bolting security onto a working system
   later, and that's where bugs come from.
3. **A device is a key, not an IP address.** Nothing keys off an address.
4. **Every command has an ID.** So a retry can't click twice.
5. **The matching logic is a plain function.** List in, action out. Testable anywhere.
6. **Never guess.** When unsure, put numbers on screen and ask.

---

## The one testing trick

Add a keystroke to the Mac app that dumps the current screen to a JSON file.

Collect a hundred — Finder, Chrome, VS Code, Slack, a dialog, a menu open, two monitors.

Now every change to the matching logic is tested against a hundred real screens in
milliseconds, and every bug a user reports becomes a permanent test.

---

## Cost

| Thing | Cost |
|---|---|
| Apple Developer Program | $99/yr, unavoidable |
| Google Play | $25 once |
| Domain | ~$10/yr |
| Vercel | Free until you take payments, then $20/mo |
| Database, auth, storage, speech | Free |
| **Year one** | **~$135, or ~$375 once you're charging** |

---

## The docs

| Doc | Open it when |
|---|---|
| **start-here** | You need the big picture |
| **technical-reference** | You're building a specific feature |
| **corner-cases** | Starting a feature. Filter to that feature only |
