# Tabletop: sUNO (v0.1, local play)

The table goes on the TV. Your hand stays on your phone.

This is the first playable slice of the platform described in [ARCHITECTURE.md](ARCHITECTURE.md): an Uno-style shedding game called **sUNO**, played around an Android TV, Google TV or tablet, with each player's Android phone as their private hand. The TV runs the game and a small server; phones on the same Wi-Fi join by scanning a QR code. Empty seats can be filled with bots.

## What's in this folder

| Path | What it is |
|---|---|
| `ARCHITECTURE.md` | The full architecture plan |
| `packages/tt_engine` | The game-engine contract and a seeded random generator (pure Dart). Every future game plugs into this |
| `packages/suno_engine` | sUNO's rules, public/private views, a bot, tests and a bot-vs-bot simulator (pure Dart) |
| `packages/tt_net` | Networking: the authoritative table host (WebSocket server, seats, lobby, bots, reconnect) and the client, plus tests |
| `apps/tabletop_app` | The Flutter app. One app for TV, tablet and phone; it adapts to the device |

## 1. Set up your computer (Windows, one time)

You need Flutter 3.47 or newer. This build was checked with Flutter 3.50.1, which ships Material as the separate `material_ui` package that this app uses.

1. Install the Flutter SDK by following [docs.flutter.dev/get-started/install/windows](https://docs.flutter.dev/get-started/install/windows). For example, unzip it to `C:\dev\flutter` and add `C:\dev\flutter\bin` to your `PATH`.
2. In Android Studio, open **Settings → Languages & Frameworks → Android SDK**. Install **Android SDK Platform 36**, **Android SDK Command-line Tools** and **Android SDK Platform-Tools**.
3. In Android Studio, install the **Flutter** plugin (it brings the Dart plugin with it).
4. Open a new terminal and run `flutter doctor`, then `flutter doctor --android-licenses` and accept them. Every Android line in `flutter doctor` should show a tick.

## 2. Build the app and install it

```powershell
cd C:\AI\Projects\MultiplayerDashboard\MultiplayerDashboard\apps\tabletop_app
flutter pub get
flutter build apk --release
```

The first build downloads Gradle and the Android dependencies, which takes several minutes. The APK lands in `build\app\outputs\flutter-apk\app-release.apk`. It is signed with the debug key, which is fine for your own devices but not for the Play Store. It contains both 32-bit and 64-bit ARM code, because many TV boxes run 32-bit Android.

**On phones:** turn on Developer options and USB debugging, plug the phone in, then run `adb install -r build\app\outputs\flutter-apk\app-release.apk`. You can also copy the APK to the phone and open it (allow "Install unknown apps"). While developing, `flutter run` with the phone plugged in installs and launches it with hot reload.

**On an Android TV or Google TV:**

1. Go to **Settings → System → About** and select **Android TV OS build** seven times to unlock Developer options.
2. In **Developer options**, turn on **USB debugging**. On newer Google TV models it is called **Wireless debugging**: choose **Pair device with pairing code**.
3. From your computer, run the following (the TV shows its IP address, ports and pairing code):

   ```powershell
   adb pair <tv-ip>:<pairing-port>    # only for Wireless debugging; type the code shown on the TV
   adb connect <tv-ip>:<port>
   adb install -r build\app\outputs\flutter-apk\app-release.apk
   ```

4. Tabletop appears in the TV's apps row.

## 3. Play

1. **Open Tabletop on the TV.** It becomes the table: a carrom board with a QR code in the centre circle.
2. **Open Tabletop on each phone**, pick a name and avatar, tap **Join a table** and scan the code. The phones must be on the same Wi-Fi as the TV. If the camera won't cooperate, type the address and room code shown under "No camera?" on the TV.
3. **The first person to join is the table captain.** From their phone they can turn on the stacking house rule, add bots and start the game. The TV remote can start the game and add bots too.
4. **Playing a card:** cards you can play sit raised. Tap one to lift it, then tap it again or flick it up towards the TV. When nothing matches, tap **Draw a card**. If the card you draw is playable, you can play it or tap **Keep it**.
5. **Calling sUNO:** with two cards left on your turn, a **sUNO!** button pulses. Tap it before playing your second-to-last card. If someone forgets, everyone else gets a **Catch them!** button, and whoever catches first makes them draw 2.
6. **Between rounds:** the round winner scores the cards left in everyone else's hands. The captain deals the next round, and scores carry over.

There's no TV needed to try it. On a tablet (or a spare phone), tap **Host on this screen**; it becomes the table. Join from another phone and add a couple of bots.

### The rules as built

The base rules are always on. The 108-card deck is dealt seven cards each, and the first card turned up is always a number. You match the top card by colour, number or symbol, or play a wild. Skip, reverse (which acts as a skip with two players), +2, wild and wild +4 all work as usual. A wild +4 can be played at any time, with no challenge. The sUNO call and the 2-card catch penalty apply, and scoring is numbers at face value, action cards 20 and wilds 50.

The one lobby toggle is the **stacking** house rule. A +2 or +4 can go on a +2, and a +4 on a +4. The total lands on the first player who can't or won't stack.

## 4. For developers

```powershell
cd packages\suno_engine; dart pub get; dart test; dart run bin/simulate.dart 5000
cd ..\tt_net;            dart pub get; dart test
cd ..\..\apps\tabletop_app; flutter test; flutter analyze
```

The simulator plays thousands of bot-vs-bot rounds. After every single action it checks three invariants: all 108 cards are still accounted for, no player's view contains another player's cards, and every legal action is accepted. A failure prints the seed so the round can be replayed exactly.

**How the pieces map to the architecture:**

- **Authority:** the TV is authoritative. `TableHost` (in `tt_net`) runs in its own isolate (`services/host_service.dart`), so animations on the board can never delay a reply to a phone.
- **Privacy:** the host sends each phone only its own private view. The board, which connects over loopback like any client, gets only the public view.
- **Instant feel:** phones play optimistically. A thrown card leaves the hand immediately and springs back if the host rejects it. The TV animates each card arriving from the seat that played it.
- **Low latency:** `MainActivity.kt` holds Android's low-latency Wi-Fi lock during a game and asks the TV for game mode (minimal post-processing). It also provides the haptic patterns and reads `tabletop://join` links, so scanning the QR code with the system camera opens the app.
- **Adding the next game:** implement `GameDefinition` and `GameBot` from `tt_engine` in a new pure-Dart package, then add board and controller widgets. `TableHost`, `TableClient`, the lobby, reconnects and bots work unchanged.

## 5. What this first slice simplifies

These are deliberate shortcuts for v0.1. Each has a planned upgrade in ARCHITECTURE.md.

| Area | Architecture plan | v0.1 | Why it's fine for now |
|---|---|---|---|
| Wire format | Protocol Buffers | JSON with the same message shapes | Messages are 1–2 KB on a LAN; the codec can be swapped later without touching game code |
| Transport security | `wss://` with the certificate pinned via the QR code | `ws://` on the LAN, gated by the room code | Fine at home. On shared Wi-Fi a snooper could read hands, so this is the next security step |
| Join link | An `https://` App Link with secrets in the URL fragment | A `tabletop://join?...` link | Needs no web domain yet. The in-app scanner and the system camera both open it |
| State sync | Snapshot plus deltas, with a replay buffer | The full redacted view on every change | Simpler and self-healing; a reconnecting phone is instantly correct |
| Remote players | Cloud relay and invites | Not yet | Local play first |
| Host crash recovery | SQLite action log | Not yet (phones do reconnect and resume their seat) | The engine is deterministic, so it is easy to add |
| "Who called first?" | Clock sync plus arbitration window | First press to reach the host wins | Fair on a LAN; ping RTT is already measured for later |
| Not built yet | | Sounds, the face-down privacy shield, a foreground service for locked phones, the "tables nearby" list (mDNS), tutorials | |

## 6. Troubleshooting

**"Couldn't reach the table."** Put the phone on the same Wi-Fi as the TV, not a guest network. Some routers block devices from talking to each other (AP or client isolation): turn that off, or have the TV join one phone's hotspot. A VPN on the phone also blocks it. The table listens on TCP port 47777, or the next free port.

**The TV says "Not on Wi-Fi".** Connect the TV to Wi-Fi or Ethernet, then reopen Tabletop.

**A phone locked or switched apps.** It reconnects by itself when you come back. After 45 seconds away, a bot plays your cards until you return.

**The font looks plain.** The Baloo 2 font downloads once on first launch and is then cached; offline devices use the system font. To bundle it instead, add the font files under `assets/fonts`, then set `Fonts.useGoogleFonts = false` and `Fonts.family` in `lib/design/tokens.dart`.

**Fire TV.** It should work by sideloading, since LAN play doesn't need Google services, but it hasn't been tested.

## 7. Before publishing anywhere

**The name.** "sUNO" still contains UNO, a Mattel trademark, and "Suno" is an existing brand. Pick a distinct name before any store listing. The card art and colour names (Mirchi, Haldi, Neem, Neel) are already original. This isn't legal advice, so check with an IP lawyer.

**Store setup.** You'll also need a release signing key, a privacy policy, and Play's TV-quality review for the Android TV listing.

## 8. What was checked in this build

All four packages pass `dart analyze` / `flutter analyze` with no issues. There are 46 automated tests: 6 for the RNG, 26 for the rules, 8 that play over real sockets, and 6 for the app widgets and host isolate. On top of the tests, 5,000 extra simulated rounds ran with no invariant failures.

On the Android side, `MainActivity.kt` was compiled against Android SDK 36 and Flutter's embedding with no errors or warnings. The manifest and resources were linked with `aapt2`, and the app registers in both the phone launcher and the Android TV launcher.

The one thing not yet run end to end is the full Gradle APK build, because the build sandbox couldn't download Gradle. It is the standard Flutter template with two small edits (`minSdk = 26`, and no NDK pin), so `flutter build apk` on your machine is the first real build. If it reports anything, paste the error and I'll fix it.
