# iPhone Duo (the folding iPhone): where the work stands

This fork makes the iOS app follow an iPhone Duo's two screens. This page is the record of what
is done, what is known to be wrong, and what is left, so the work can be picked up later.
Last updated 2026-11-14.

## The goal

The game is played held sideways, folded or unfolded, and should fill either screen sharply.
Folding or unfolding in the middle of a game should not leave bars or a soft picture.

## What the phone does (measured on a real Duo and in the simulator)

- Outer (folded) screen: 1398x2034 pixels at 3x, 466x678 points. It honours the app's landscape
  lock. The Dynamic Island gives a safe area of 84 points on one side.
- Inner (unfolded) screen: 2007x2853 pixels at 3x, 669x951 points. It ignores the app's landscape
  lock: the scene follows the device through all four orientations (951x669 or 669x951 points).
  Its left and right safe areas are 0 when held sideways.
- The hinge is on the long edge: a phone held sideways while folded opens into a tall scene.
- The app runs in the phone idiom on both screens. iOS does not letterbox it, because the
  release is built with a recent SDK and the app adopts the scene lifecycle.
- An unchanged upstream build already fills whichever screen it is launched on. What it does not
  do is follow a fold: its surface keeps the size it had at launch.

## Done (on `main`)

- `5e3e351` The surface follows the screen. On a phone, when the scene settles at a new size the
  host gives the game a new surface in landscape order (`surface_follow_scene` in
  `runtime/tools/pvz2-ios.m`): half a second of debounce, never before `tl_pvz2_started()`, and
  only for shapes between 4:3 and 21:9.
- `c6829bc` The game is told its new shape. On iOS the driver's access to the game's app object
  was a stub; it now reads and writes it in its own task and finds it through the game's own
  pointer (the app getter's GOT slot), so the game's app size follows the surface
  (2234x1536 folded, 2183x1536 unfolded).

Checked on a real Duo: launched folded and launched unfolded, the picture is drawn at the
screen's own pixels at scale 1.0 with no bars. An unfold in mid-game resized the surface and the
app size as the log lines above say.

## Known problems

1. **The safe-area margin is read once.** The game asks for the cutout insets one time, while it
   starts (`J_safe_rect` in `husk-tl-pvz2.c`, "The game asks once"). Launched folded, it keeps
   252 pixels clear for the island; after an unfold the host reports 0, but the game never asks
   again. The level's top row then sits too far right and the pause button runs off the screen.
   Launched unfolded and then folded, the opposite is expected: nothing kept clear of the island.
   Until this is fixed: open the game in the shape it will be played in, and reopen it after
   switching.
2. **The screen that is open during a fold keeps its old layout.** Screens opened after the
   change are laid out for the new shape. The one showing at that moment is not: in the
   simulator the main menu kept a 45 pixel black column after a change to the wider shape.
3. **Unfolded and held tall, the game is a small strip.** The inner screen hands the app portrait
   scenes, and the host still draws a landscape strip with bars in them. See "Left to do".
4. **A loading screen that never finished, once.** On the real phone, in a session launched
   folded and unfolded 54 seconds in, a loading screen started about 611 seconds in and was still
   showing when the app was closed 78 seconds later. The game kept drawing at full rate the whole
   time, and there was no crash. Reopening the game continued normally. The cause is not known,
   and it is not known whether the fold has anything to do with it.

## Left to do

- **Tell the game the new insets after a fold** (problem 1). The value is cached inside the
  game, so this needs the place it is kept, or the call that reads it again.
- **Make the open screen lay out again** (problem 2). An experiment was started and not
  finished: after the surface change, call the game's own `LawnApp::SetWidthHeight` (the recipe
  the driver's orientation path documents above `GUEST_DRIVER_GLOBAL`), or
  `Native_onOrientationChanged`, or a surface re-create, or a background and foreground, and
  measure which one makes the open screen lay out again. No result was recorded for any of them.
- **Keep the picture along the screen's long side** (problem 3). In a portrait scene on a phone,
  turn the game's view a quarter turn so it fills the scene and does not move on the glass when
  the phone is turned; map touches, safe insets and the host's overlays through the turn. Two
  routes: `prefersInterfaceOrientationLocked`, or a transform on the game's view.
- Reviews of the whole change, and an offer of the work upstream, have not happened.

## Testing without a phone

The iPhone Duo simulator can be folded and turned only by hand, with the buttons under its
window in Device Hub: `simctl` and `devicectl` have no command for it. A stand-in exists for
scripts: launch with `TL_PVZ2_TEST_SCENE=951x669` (points) and the surface follows that size as
if the scene had settled at it. Screenshots of one screen: `xcrun simctl io <udid> screenshot
--display=1` (outer) or `--display=3` (inner).

## Building for a real Duo

    SIGN_IDENTITY="<an Apple Development identity>" SIGN_PROFILE="<a development profile that lists the phone>" \
        ./build.sh --ios --quick --apk <the 1.4.2 apk> --obb <the obb>
    xcrun devicectl device install app --device <udid> out/.work/ios/PvZ2.app

A wildcard development profile works with the app's own bundle ID. With `--obb` the game data is
inside the app, so the first launch downloads nothing. The game's log is `Documents/pvz2.log` in
the app's data container, and the session before it is `Documents/pvz2.previous.log`.

## Keeping up with updates

Two things are updated, on different clocks (as of 2026-11-14):

- **The port** (upstream, `oenderg/Reflourished-Darwin`): six releases in its first three weeks.
- **The game** (the mod itself): a new version every few months; 1.4.2 is from April 2026.

A port update on this fork: merge upstream's `main` into this fork's `main`, build as above and
install over the app on the phone. The app ID and the signing team stay the same, so the saved
game stays. The fork's code is two commits in two files (`pvz2-ios.m`, and iOS-only code in
`husk-tl-pvz2.c`); upstream changes the driver often, so a merge can conflict there.

A new game version needs upstream to support it first ([NEW-GAME-VERSION.md](NEW-GAME-VERSION.md)).
This fork adds one game address of its own to the list that must be found again when
`libPVZ2.so` changes: the app getter's GOT slot that the iOS driver reads the app object from
(`guest_app_find`). `libPVZ2.so` was identical between 1.4.0 and 1.4.2.

If a merge is ever not worth it, upstream's own release fills whichever screen it is launched
on; it only does not follow a fold in mid-game.

## Keeping the save in step with a Mac

The progress is one file, `pp.dat`, in
`<data>/sdcard/Android/data/com.ea.game.pvz2_rfl/files/No_Backup/` (the game also keeps
`pp.dat.bak` and falls back to it). `<data>` is `Documents/data` in the iOS app's container and
`~/Library/Application Support/PvZ2 Reflourished` on the Mac. Measured on 2026-11-14:

- A phone's `pp.dat` copied to the Mac as `pp.dat` and `pp.dat.bak`, with the game closed, loads
  there: same profile, level and coins. The game does not rewrite the file unless progress is made.
- A save dropped in as `pp.dat.imported` (what Import Save does) loads too, but the game consumes
  it and writes no `pp.dat` until progress is made, so a copy as `pp.dat` is the safer way.
- A Mac can read and write the files of a development-signed iOS app over Wi-Fi:
  `xcrun devicectl device copy from|to --domain-type appDataContainer --domain-identifier <app id>`.
  A request fails now and then and works on the next try.
- A whole `pp.dat` starts with `RTON` and ends with `DONE`.

A small script outside this repository uses this to sync before and after a Mac session. Not yet
checked: the phone loading a save that the Mac changed. Quests live in other files.
