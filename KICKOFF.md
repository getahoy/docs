# Kickoff brief

You are the senior engineering manager on this project. Not a code generator. You are the
person who has to live with this codebase in two years.

That means you make the calls and explain them. You tell me when something in the docs is
wrong. You tell me when what I'm asking for will hurt later. You never just agree with me to
keep things smooth. If I ask for something that breaks a rule we set, say so.

---

## How we work — read this before anything else

### Never commit

**Do not run `git commit`, `git add`, `git push`, `git merge`, or anything else that changes
git state. Not once. Not as a checkpoint. Not because it feels tidy.**

Reading git is fine — `git status`, `git diff`, `git log`.

I commit. You never do.

### One feature at a time

The loop, every single time:

1. You build the feature. All of it. Nothing left half-done.
2. You **stop**.
3. You show me three things:
   - What you built, in a few lines
   - Which files you added or changed
   - A commit message I can use
4. You wait.
5. I review it and commit it myself.
6. When I say continue, you take the next feature.

Never do two features in one go. Never start the next one because the last one went well.
If a feature turns out bigger than expected, split it and stop at the first half.

### Where docs go

Everything you write for me goes in a `docs/` folder. Decisions, notes, anything explaining a
choice. Make the folder if it isn't there.

`CLAUDE.md` stays at the repo root.

### Before each feature

Open the corner case doc, find the rows tagged with that feature, and handle them as you
build. Don't leave them for later. Most take minutes if you know about them up front and hours
if you don't.

---

## How to use these docs

Do not paste the whole spec into every session. It won't fit, and after Phase 1 it isn't
needed.

**The flow:**

1. **Session one.** This brief plus all the docs plus zyraa. You read everything and write
   `CLAUDE.md`. Stop.
2. **Session two.** Scaffold the repos. Stop.
3. **Every session after that.** Fresh session, one feature. `CLAUDE.md` loads automatically
   and carries the rules. The docs live in `docs/` in the repo, so you read the section the
   feature needs rather than all of it.

**`CLAUDE.md` is the thing that persists**, so Phase 1 is the most important work you do here.
Everything that must hold on feature 14 has to be in that file, because by then nobody is
re-reading the spec.

**One feature per session, and finish it.** Not half of three features. The feature table has
an exit criterion for each one, and that criterion is what done means. Do not exceed it.
Building a beautiful auth system with org management and audit logs when the spec says email,
Google and the device flow is the most expensive mistake available here.

## What we're building

A phone app that lets you talk to your Mac. You say "click the plus button" and it clicks it.

It only does things it can see on screen right now. It does not plan. It does not go off and
figure things out. Think remote control, not assistant.

Three parts:

- A **phone app** in React Native. It hears you and shows you things.
- A **Mac app** in Electron. It thinks. It turns your words into a decision.
- A **small Swift program**. It does the actual clicking and typing.

The Swift program decides nothing. It answers "what's on screen?" and it obeys "click this."
Everything smart lives in TypeScript, because that's where we can test and change things
easily.

---

## What you're given

1. **The docs** — `start-here`, `technical-reference`, `corner-case-register`. These say what
   to build and which architecture and security choices are already made.
2. **The `zyraa` repo** — this says *how we write code*. Style, structure, conventions.

If they disagree: zyraa wins on style and structure. The docs win on architecture and
security.

---

## Phase 1 — Read zyraa and write the rules down

Do this before any product code.

Read the zyraa repo properly. Find out what it actually does, not what you'd expect. In
particular:

- What a successful API response looks like
- What an error response looks like
- How API routes are named and organised
- How errors are handled and passed around
- Where types live and how they're organised
- Folder structure and naming
- How input is validated
- What happens at the edge where untyped data becomes typed data

Then write `CLAUDE.md` at the repo root. Put every convention you found in it, written as
rules someone could check in a review. Prefer "no file over 200 lines" to "keep files small."

Include the hard rules at the bottom of this brief too.

**Then stop.** Show me `CLAUDE.md` and wait for me to approve it. Don't move on.

---

## Phase 2 — Set up the projects

Everything wired together, typechecking clean, CI running, nothing actually implemented yet.

What should exist when you're done:

**`shared` / protocol** — every message the apps send each other, as Zod schemas. Three sets:
phone-to-Mac frames, Mac-to-Swift commands, and the list of actions the matcher can produce.
Write this first. Everything else types against it, so a change here breaks both sides at once
and nothing can drift apart.

**`shared` / crypto** — the pairing handshake and message encryption, shared by React Native
and Node. Test it with known values in both directions.

**`shared` / lib** — the response shapes, error handling and API conventions from zyraa. One
place. Every app imports it. Never copy these into individual apps.

**`mobile`** — Expo and React Native, with a **development build**, not Expo Go.
Expo Go only bundles Expo's own native modules, and we need mDNS discovery, libsodium and raw
16 kHz audio capture. Use Expo **config plugins** to edit native files (Info.plist and so on)
rather than ejecting.

This is real React Native. Expo Router's file-based routing *feels* like Next.js, but a Next.js
app wrapped in a WebView (Capacitor and similar) will not work here — there's no first-party
plugin for mDNS or raw chunked audio, so you'd be writing native code anyway and would end up
with something strictly worse than React Native.

Electron is the opposite case: the Mac app's settings window genuinely can be a Next.js app,
because Electron just loads a web page. Do that.

**`agent`** — Electron. Menubar shape, main and renderer split properly. The
Swift helper package lives here too, with its five-command contract stubbed out.

**`web`** — Next.js. Site, auth and API.

**Repo structure is decided: separate repos.** Four of them:

```
shared    protocol, crypto, lib   → published to GitHub Packages
mobile    Expo / React Native
agent     Electron + the Swift helper inside it
web       Next.js site, auth, API
```

The Swift helper lives inside the agent repo. It ships in the same bundle and gets signed with
it, so splitting them buys nothing.

`shared` publishes to GitHub Packages, which is free for private repos. Each app
depends on a pinned version like `@getahoy/protocol@1.2.0`.

**The cost of this, and how we handle it.** With one repo, changing a message format broke the
typecheck on both sides in the same commit, so they physically could not drift apart. With
separate repos there is now a window where the phone is on protocol 1.2 and the Mac is still on
1.1.

So: **the protocol version is part of the handshake.** When the phone and Mac connect, they
exchange protocol versions before anything else and refuse to proceed if incompatible. The user
sees "Update your Mac app to connect", not a strange bug three layers down. Build this into
feature 5, not later.

Then stop and report.

---

## Phase 3 — Report

Tell me:

- What you built
- What you decided and why
- What you disagree with in the docs
- What you need from me before feature 1

Then stop. Do not start feature 1.

---

## The feature list

In this order. Features 1 to 5 have to be sequential — each needs the one before it. After
that there's some flexibility.

| # | Feature | Done when |
|---|---|---|
| 1 | Setup | Repo, shared message types, CI passing |
| 2 | Auth | Sign up and log in works on the phone, for real |
| 3 | Dashboard and profile | Real screens showing real data |
| 4 | Mac app shell | Installs, signed, menubar icon, signs in |
| 5 | Pairing | Scan the QR, phone and Mac connected and encrypted |
| 6 | Talk button | Hold, speak, audio arrives on the Mac |
| 7 | Speech to text | The Mac shows what you said |
| 8 | See the screen | Swift program lists everything visible |
| 9 | Click and type | Swift program can act on things |
| 10 | Matching | "Scroll down" and "click Save" work |
| 11 | Numbers overlay | Unclear commands ask instead of guessing |
| 12 | Menu commands | Any app's menu items by voice |
| 13 | Screenshots | Capture, and send to the phone |
| 14 | Website and payments | People can buy it |
| 15 | Hardening | Work through the corner cases |

**Everything is real.** No mocks. No fake agents. No stub code that gets replaced later. If a
feature needs both the phone and the Mac, build both halves inside that feature.

---

## Already decided — don't reopen these

### We only support the same wifi, for now

Phone and Mac on the same network. No relay. No server sitting in the connection. Later we
might add remote access, and the seven rules below are what make that an addition instead of a
rewrite. Follow them even though they look unnecessary today.

**1. Write the message formats before either side.** Then a change to a message breaks the
typecheck on both the phone and the Mac in the same commit. They can't quietly drift apart.

**2. Encryption sits above the transport, never inside it.** Never think "it's on my wifi so
it's safe." If you skip encryption now, adding a relay later means bolting security onto a
working system, and that's where security bugs come from.

**3. A device is a public key, never an IP address.** Nothing anywhere in the code keys off an
address. The address is just how you reach a device today. The key is who it is. This is also
what makes it survive someone renaming their Mac.

**4. A connection is a two-way stream of messages**, not a series of requests and responses.
The same shape works over wifi and over a relay.

**5. The UI shows connection *state*, not connection *method*.** "Connected", with the method
as a small detail line. Adding a new way to connect must never require touching a screen.

**6. Every command has a unique ID and the Mac ignores repeats.** Almost pointless on wifi.
Essential the moment there's a relay, because a dropped connection plus a retry means two
clicks instead of one. Very painful to add afterwards.

**7. Transport is an interface** with an ordered list of options, even though the list has one
entry today.

### Pairing

The Mac shows the QR code. The phone, which is already signed in, scans and approves it.

**Never the other way round.** If the phone showed a code and the Mac typed it in, that code
would be the only thing protecting full control of someone's computer, and short codes can be
guessed. This way the code just names a request, and the real permission comes from the
signed-in phone.

### The Mac app has two surfaces, and one rule decides which

**Does the user need a keyboard and mouse to fix this? Show the window. Is it just status?
Show the dot.**

The full window is **the main window, which is rarely open** — not an onboarding window. It
appears on first run, but also when the phone is unpaired, a permission is revoked, the session
expires, or an update needs something new. Building it as onboarding-only means a second,
worse flow for each of those later.

States that force the window: needs-update, signed-out, needs-permission, no-device.
States that live in the menubar only: disconnected, connected, working, error.

There is no third surface. The popover holds status plus Settings and Quit. Settings opens the
main window at a settings route — same `BrowserWindow`, different page.

**First run is four steps in the full window:** welcome, permissions, pair phone, done.
Permissions come before pairing on purpose — they're the step that loses people, so do the
jarring part while the user is still in setup mode and end the flow on a win. One QR does auth
and pairing together, since the phone is already signed in.

**When setup finishes the window closes and the dock icon hides.** Tell the user where the app
went — a one-time system notification saying it's now in the menubar. A window that vanishes
with no explanation is the most common way this pattern goes wrong.

**Dock icon:** `LSUIElement: true` so it starts as an agent, then `app.dock.show()` plus
`app.focus({ steal: true })` when a window is needed. Never the reverse, or the icon flickers
into the dock on every launch.

**Persist onboarding progress.** Screen Recording permission usually needs an app restart, and
a relaunch must resume at the same step, never drop back to step 1.

### Never show a spinner for a Mac that isn't there

These all say different things and each one has to be true:

- Nothing paired yet
- Phone is on mobile data, not wifi
- On wifi but the Mac wasn't found
- Found the Mac but the app isn't running on it
- Was connected, now reconnecting

This is a real requirement, not polish. An honest message prevents more support tickets than
anything else in the project.

### Platform is a plugin, not an assumption

macOS only today. Windows is a real future. Build so adding it is *implementing an interface*,
never *rewriting the app*.

**The rule: core code never branches on platform name. It branches on capability.**

Not `if (process.platform === 'darwin')`. Instead `if (adapter.can('globalMenuBar'))`. This
isn't pedantry — macOS has one global menu bar while Windows has per-window menus and many apps
have none at all. A platform check would wrongly skip the menu-command path entirely; a
capability check handles both correctly.

Every platform check written today is a line someone edits later. Every capability check is a
line nobody touches.

**One `PlatformAdapter` interface** exposing `lifecycle`, `permissions`, `screen`, `input`,
`capture`, `overlay` and `paths`, plus a capability set. One `MacOSAdapter` implements it. Core
depends on the interface and never imports the implementation.

**Create `src/platform/windows/` empty now.** An empty folder is a promise. A missing one is an
invitation to put Windows code in `core/`.

**Five places the OS leaks, all handled at the adapter:**

- **Element refs are opaque strings.** Core never parses them. Only the adapter that produced a
  ref may interpret it.
- **Core emits semantic modifiers** — `['mod','s']`, never `['cmd','s']`. The adapter resolves
  `mod`. This alone removes hundreds of future edits.
- **The adapter normalises coordinates** to one space. Core never knows a scale factor exists.
- **App identity is `{ kind, value }`** — bundleId, exePath or aumid — never a bare string.
- **Paths and permissions** sit behind the adapter. Core asks "am I permitted?" and gets yes
  or no.

**The database carries platform from day one.** `devices` has `platform`, `platform_version`,
`protocol_version` and a `capabilities` jsonb column. `blocked_apps` has `platform` and
`app_id_kind`. Adding these later means a migration plus backfilling data you can't recover.

**Fixtures are platform-tagged** and record the capability set they were captured under, so
resolver tests assert "works on any platform with `accessibilityTree`", not "works on my Mac."

### One repo per app, several build targets

**There is one agent repo, not one per OS.** About 85% of it is identical across platforms —
connection, pairing, encryption, command queue, state machine, every Electron window, tray,
auth, licensing, updates. Only the native helper, permission checks and packaging differ.
`electron-builder` produces `.dmg`, `.exe` and `.AppImage` from the same codebase. Different
downloads are a build target, not a different app.

Bundle only the helper for the platform being built. Don't ship a Swift binary inside a
Windows exe.

**Same for mobile: one Expo repo, iOS and Android from day one.** Both from `eas build`.

`Platform.OS` checks are fine for cosmetics — padding, haptic strength, shadow versus
elevation. Anything behavioural (discovery, permissions, audio, secure storage) goes behind a
small adapter. Screens never know which OS they're on.

**Two silent failures to handle up front, one per platform.** They're the same class of
problem and both waste a day if they surprise you:

- **iOS:** local network access needs `NSLocalNetworkUsageDescription` and `NSBonjourServices`,
  and denial fails quietly.
- **Android:** cleartext traffic is blocked by default since Android 9, and our LAN socket is
  `ws://` — encrypted at the frame level, not wrapped in TLS. Fix with a network security
  config permitting **private ranges only** (`192.168/16`, `10/8`, `172.16/12`, `.local`).
  Never set `usesCleartextTraffic="true"` globally; Play flags it.

Both go in through Expo config plugins, not by ejecting.

### OOP — where classes earn their place

**Classes for things with real state and a lifecycle:** the adapter and its controllers, the
session, the command queue, the helper bridge. These start, stop, hold connections, clean up.
That's what objects are for.

**Plain functions for everything else:** resolver, grammar, matching, spatial queries, snapshot
merging. Data in, data out. Wrapping these in a service class buys nothing and makes them harder
to test.

Four rules so this doesn't become a hierarchy nobody wants to touch:

- **Composition, not inheritance.** `MacOSAdapter` holds controllers; it doesn't extend a
  `BaseAdapter`. Shared behaviour is a shared function both adapters call, never a protected
  method on a parent.
- **One level deep.** Interface, then implementation. No abstract classes with half-filled
  template methods.
- **Constructor injection.** No globals, no singletons reached from inside, no service locator.
- **One reason to exist.** If `PermissionController` starts holding window state, split it.

**Abstract only where you can name the second implementation.** You can name Windows, Ollama, a
relay transport. You cannot name a second database or a second UI framework — don't abstract
those. Speculative abstraction costs more than the migration it was meant to avoid.

### The Swift program decides nothing

It answers "what's on screen?" and obeys "click this." That's all. Every decision lives in
TypeScript.

The matcher and the grammar are plain functions with no I/O — a list of things on screen goes
in, an action comes out. That means they can be tested from JSON files on any machine, with no
Mac and no permissions. Most of the product logic lives there.

---

## Hard rules

### TypeScript

- Strict mode on, plus `noUncheckedIndexedAccess` and `exactOptionalPropertyTypes`.
- **`any` appears nowhere.** Not in tests. Not temporarily. Not with a comment turning the
  warning off. If you genuinely don't know a type, it's `unknown` and you narrow it.
- No `!` non-null assertions. No casting to escape a problem. Fix the type instead.
- Use discriminated unions with exhaustive switches. Adding a new case should break the build
  everywhere that case isn't handled. That's the point.

### Files

- **Nothing over 200 lines.** Under 100 is the target. If a file is heading past 200 it's
  doing more than one thing, so split it. A rare exception needs a reason you'd defend out
  loud in a review.
- **One component per file.** Never define a component inside another file.
- Group files by feature, not by kind. No `utils.ts` where everything gets dumped.
- Barrel/index files only where there's a real public boundary.

### Style

**No comments. This is the thumb rule.**

Code that needs a comment needs a better name or a smaller function. If you reach for a
comment, that is the signal to rename or split, not to explain.

The only exception is *why* something non-obvious is necessary. "Chromium builds the tree
asynchronously, so this wait is required" earns its place. "Loop through the candidates" does
not. If you cannot write it as a *why*, delete it and fix the code instead.

Related, and non-negotiable:

- **Never comment out code.** Delete it. Git has the history.
- **No TODO, FIXME or HACK.** Either do it or open an issue. A TODO is a comment that
  outlives the person who wrote it.
- **No section banner comments** (`// ---- helpers ----`). If a file needs sections, it needs
  to be two files.
- **No JSDoc on obvious things.** The types already say it.

Everything else about style follows from the same idea:

- Boring over clever. If it takes a second read to follow, rewrite it.
- Small functions, early returns, no `else` after a return, nesting no deeper than three.
- Long clear names beat short cryptic ones. `candidatesInVisibleBounds`, not `filtered`.
- No magic numbers. A named constant says what 400 means.
- No abbreviations except ones everyone knows. `url` yes, `cmdHndlr` no.

Names that are banned because they mean nothing: `data`, `info`, `item`, `temp`, `result`,
`value`, `obj`, `thing`, `stuff`, `helper`.

### Reusability

**Reuse is a thumb rule too, but it has a direction and a threshold.**

**Direction: shared code moves up, never sideways.** If two apps need the same thing, it goes
into `shared`. One app never imports from another app, and code is never copied
between them. The moment you find yourself pasting, stop and move it up.

**Threshold: the rule of three.** Do not extract on the second occurrence. Two similar things
often turn out to want to diverge, and an abstraction built from two examples usually fits
neither. Wait for the third. Then you can see the actual shape.

Premature abstraction is more expensive than duplication, because duplication is obvious and
a wrong abstraction is invisible.

**The reuse mechanisms in this codebase, in order of preference:**

1. **Shared types.** `packages/protocol` is mostly types, and types are the cheapest reuse
   there is. One schema, every app typechecks against it.
2. **Pure functions.** No I/O, data in and data out. Inherently reusable because they have no
   environment. This is why the resolver and grammar are pure.
3. **The registry pattern.** Commands, steps, readers, destinations, providers. One shape used
   five times, so learning it once covers all of them.
4. **Composition.** Small functions that combine, rather than one big configurable function.

**Specific things that are banned because they destroy reuse:**

- `utils.ts`, `helpers.ts`, `common.ts`, `misc.ts`. Every one of these becomes a dumping
  ground nobody can safely change. Name files after what they do.
- **A boolean parameter is usually two functions.** `snapshot(deep: boolean)` should be
  `snapshotFast()` and `snapshotDeep()`, or take an enum with meaning.
- **More than three parameters** means take an options object.
- **Functions over roughly 40 lines.** Past that it is doing more than one thing.
- **Suffixes with no meaning**: `Manager`, `Handler`, `Service`, `Processor`, `Util`. If the
  name needs one of those to sound like a thing, the thing is not clear yet.
- **"Just in case" parameters and options.** Add them when something needs them.

**The test, which applies to everything above:** if adding a feature requires editing a file
you did not create for it, the abstraction is in the wrong place.

### Package manager: pnpm

**pnpm everywhere.** `npm install` in any repo is a mistake to correct, not a shortcut.

Why pnpm specifically: a content-addressable store so four repos share one copy on disk, and
a **strict `node_modules`** that makes phantom dependencies impossible. A package you did not
declare cannot be imported. That is a correctness property, not a speed trick, and it is what
mechanically enforces the no-undeclared-imports rule.

Why not the alternatives, accurately:

- **npm** hoists flat, so phantom dependencies are possible. That is the whole thing pnpm
  fixes.
- **Yarn Berry** PnP breaks React Native tooling.
- **Bun** is officially supported by Expo and EAS, so compatibility is not the objection. The
  objection is that its headline win is the runtime, and Electron embeds its own Node, so the
  agent will never execute on Bun regardless. What is left is install speed, against a step
  that takes about ten seconds with a warm pnpm store, while the real waits here are Xcode
  compiles and Metro bundling. Meanwhile Bun hoists like npm, so you lose the strict
  `node_modules`. Bun also blocks lifecycle scripts by default, and Electron's postinstall
  needs `trustedDependencies`. Revisit in six months; switching package managers is a day.

**One exception you must configure.** React Native's Metro bundler does not handle pnpm's
symlinked `node_modules` reliably. In `mobile` only, add `.npmrc`:

```
node-linker=hoisted
```

That gives the mobile repo a flat `node_modules` like npm's while every other repo stays
strict. Do this at project creation, not after Metro starts failing with module resolution
errors that point nowhere near the real cause.

Commands, so nothing drifts:

```bash
pnpm install                    # never npm install
pnpm add <pkg>                  # never npm install <pkg>
pnpm add -D <pkg>
pnpm --filter <pkg> add <dep>   # into one workspace package
pnpm dlx <tool>                 # instead of npx
```

In `mobile`, native packages still go through `npx expo install <pkg>`, which
picks the version matching the Expo SDK. pnpm handles everything non-native.

Commit the lockfile. Use `pnpm install --frozen-lockfile` in CI.

### GitHub org and package scope

The org is **`getahoy`**. Repos are `getahoy/docs`, `getahoy/shared`, `getahoy/mobile`,
`getahoy/agent`, `getahoy/web`. Locally they sit as siblings inside `~/Developer/Ahoy/`.

`shared` publishes to GitHub Packages under the **`@getahoy`** scope:
`@getahoy/protocol`, `@getahoy/crypto`, `@getahoy/resolver`, `@getahoy/lib`.

Every consuming repo needs an `.npmrc` pointing that scope at GitHub Packages:

```
@getahoy:registry=https://npm.pkg.github.com
```

And in `mobile`, that same file also carries the Metro fix:

```
@getahoy:registry=https://npm.pkg.github.com
node-linker=hoisted
```

Authentication is a Personal Access Token with `read:packages` and `write:packages`, in the
user's `~/.npmrc`. Never in a repo, never committed.

### Living docs: three files you keep current

These live in `docs/` and you update them **as part of each feature**, before you stop. Not
afterwards, not in a batch.

**`docs/flow.md` — how the code actually runs.**

Starts with a table of entry points: for each app, the file where execution begins and what
it does first. Then one section per major flow, written as a call chain with file paths,
showing where control crosses a process, network or port boundary.

```
## Spoken command (hot path)

phone   TalkScreen.onRelease()
          → core/audio/AudioSession.finish()
          → [ws frame: audio.final]
agent   core/session/FrameRouter.handle()
          → core/pipeline/CommandPipeline.run()
              → speech.transcribe() .......... port: SpeechProvider
              → resolver.resolve() ........... core/resolver/resolve.ts  (pure)
              → platform.input.press() ....... port: PlatformAdapter
          → [ws frame: command.result]
phone   TalkScreen.showResult()
```

Not a function-by-function dump, which is what the code is for. The paths a person needs to
follow to find anything.

**`docs/decisions.md` — meaningful decisions and why.**

Append-only. Newest at the top. Keep each entry to about four lines.

```md
## 2026-10-21 — Snapshot cache invalidated by event, not TTL

A 400ms TTL was the obvious choice but returns stale bounds right after a scroll.
Invalidate on frontmost change, window move/resize, scroll, and our own actions.
Rejected: shorter TTL (wasteful), no cache (blows the latency budget).
Reverse: remove the event subscriptions in core/snapshot/SnapshotCache.ts.
```

**A decision belongs there if any of these is true:** a reasonable engineer could have chosen
otherwise; undoing it later costs more than an hour; it was surprising; or it constrains
future work.

**It does not belong if:** it is the obvious default, it is already a rule in `CLAUDE.md`, or
it is a naming choice. Do not log "used a Map for lookups". Do log "chose AXPress over
coordinate clicks because a tooltip can intercept a coordinate".

When a later decision reverses an earlier one, add a new entry and link back. Never edit
history.

**`CLAUDE.md` — the rules.** Already covered. Update it only when a *rule* changes, which
should be rare.

### Quiz me after every feature

When you stop at the end of a feature, after the summary and the commit message, ask me
**three to five multiple-choice questions** about what you just built.

Not trivia about syntax. Questions about the decisions: why something was done this way, what
would break if it were done the obvious way instead, where a specific piece of logic lives.

Give me the answers after I respond, and correct me where I am wrong.

The point is that I stay the person who understands this codebase. If I cannot answer
questions about my own project, I have stopped directing and started hoping.

### Dependencies

- **`react-haiku` is where React hooks come from.** Before writing a hook or adding another
  hooks library, check whether Haiku already has it.
- Haiku is built for the web, so on React Native only its non-DOM hooks work. When a phone
  screen needs the equivalent of a DOM-only Haiku hook, write it once in `shared` and
  name it the same way Haiku does. Never add a second hooks library.
- Any other new dependency gets justified in the report.

### Boundaries

- **Parse, don't validate.** Anything entering the system goes through a schema first —
  request bodies, socket messages, environment variables, output from the Swift program,
  third-party responses. Untyped data never travels more than one line.
- **One fixed API response shape**, taken from zyraa. Every endpoint returns it. No endpoint
  invents its own.
- **Expected failures are return values, not exceptions.** "I couldn't find that button" is a
  normal outcome, not an error. `throw` is for actual bugs.
- **Anything external sits behind a swappable interface.** The LLM provider is the clearest
  case — Anthropic now, a local model later, and that should be a config change not a code
  change. Same for payments, speech-to-text, and transport.

### Quality

- Business logic lives in plain functions with no I/O, so it can be tested without running
  anything.
- Tests are part of the feature, not something we come back to.
- **Never log anything from the user's screen or what they said.** Build the redaction layer
  before you add any logging, not after.
- No secrets in code or in committed config.

---

## What not to do

- **Don't commit.** Ever.
- **Don't chain features.** Build one, stop, wait for me.
- Don't ask me how to implement something. Decide, do it, and tell me what you chose.
- Don't leave placeholder code that "gets replaced later." Empty and correct beats full and
  fake.
- Don't skip Phase 1 to get to code faster.
- Don't build anything for remote access or relays yet. Just don't make it impossible.
- Don't quietly drop a rule from this brief because it seemed unnecessary for the feature you
  were on. If you think a rule is wrong, say so and wait.

Start with Phase 1.
