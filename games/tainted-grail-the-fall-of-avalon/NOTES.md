# Tainted Grail: The Fall of Avalon: working notes

Raw notes, written as we go. The write-up (README.md) comes later.

## Game identity

| | |
|---|---|
| Store | Steam, app 1466060, beta branch `mono` |
| Build | buildid 25966682, game version 1.30.102 (`[1.30a]`), saves written by 1.15.033 |
| Main exe SHA-256 | `e01f1a4cfa8ebb93f7d5c9ea3925cef2883e1d230741ede76ad4f4c7d1bcd594` (`Fall of Avalon.exe`) |
| Engine | Unity 6000.0.41f1, Mono scripting on the `mono` branch (IL2CPP on the default branch) |
| OS | Windows 11 |

Game code: `<game dir>\Fall of Avalon_Data\Managed\TG.Main.dll` (about 6,200 decompiled files),
plus `Awaken.*.dll`. Decompiled with ilspycmd 11.1.

## Existing work

- **TGCoop** ("2 Players Steam Coop"), https://www.nexusmods.com/taintedgrailthefallofavalon/mods/211.
  BepInEx 5 + Harmony, 2 players, Steam lobbies and SteamNetworkingMessages on the game's own app
  id, host authority. Latest build seen: 0.5.36 (2026-07-28). No license file. Author's last reply
  on Nexus 2026-07-14; later comments report softlocks and no fixes.
  - Session model (from its readme): both players load their own save, then the guest joins.
  - Story sync (from reading the decompiled DLL): live hooks on `StoryFlags.Set`,
    `QuestUtils.SetQuestState`, `QuestUtils.ChangeObjectiveState`; on join the host sends only
    the true boolean facts of each memory context.
  - Bugs reported by players (Nexus comments, July to October 2026): gate stays closed for the
    guest after the Odrán dialogue; guest frozen by dialogue during boss fights; combat desync;
    A-pose unless both players use third person; mod dialogue box covers the game's choices.

## How the game works (for co-op)

### Observations (from the decompiled code)

- Own MVC framework (`Awaken.TG.MVC`): `Model` with string IDs `<Type>:<counter>`, nested
  elements `<parentId>:<Type>:<counter>`, counters per type in `IdStorage`, events through
  `ModelExtensions.Trigger`, services through `World.Services`.
- Story state is a key/value store: `GameplayMemory` → `ContextualFacts.Set/Change/Remove`.
  `StoryFlags.Set`, quest state (`Context(quest).Get("state")`) and objective state all live there.
  Other writers include `ActorsRegister` (`Context("actors")`), `Location`
  (`Context("discarded.places")`), enemy behaviours, story variables, visited scenes.
- One player everywhere: `Hero.Current` is static and referenced in about 1,240 places.
- Saves: one zip per slot, one `.data` file per domain (`Gameplay.data`: hero, items, quests,
  memory; one file per visited scene; `MetaData.data`). All saves and prefs go through
  `CloudService` (Steam Cloud, Steam without cloud, GOG, debug).
- Models are saved and restored in world registration order, in one `RestoreWorld` pass. While
  restoring, world quests read hero stats (`StatTracker`), and hero elements read world models
  (`TrespassingTracker` → `GameRealTime`), so hero and world depend on each other at load time.
- New Game+ (`NewGamePlusUtils`) already extracts a character (hero subtree, hero and stash items
  minus quest items, keys and readables, journal, achievement quests, game time) into a blob and
  restores it into another world (`LoadSystem.DeserializeNewGamePlusCache`).
- Difficulty is a setting keyed by the character's `HeroID` GUID in PlayerPrefs, not part of the
  save slot. Story, Easy, Normal, Hard, Survival (save restrictions), Challenge (locked, adds a
  `HeroChallengeMode` element to the hero).
- The game ships Steamworks.NET + Heathen, GOG Galaxy, Unity User Reporting (automatic bug
  reports with recent saves, logs and settings attached when loading fails), Sentry, GameAnalytics,
  and its own `Automations` hook on the title screen.

### Character transfer experiment (2026-10-18)

Goal: put player B's character into player A's world, so a co-op guest keeps their character.
Method (BepInEx plugin, test install only):

1. Export B's character with the New Game+ selection and serializer.
2. Load A's save normally. Right after the Gameplay domain is restored, discard A's hero and its
   items, then deserialize B's blob. While reading the blob, every `Item:<n>` ID is offset by
   100000 (hooked in `SaveReader.ReadAscii`, which all IDs pass through) so B's items cannot
   collide with A's world items; B's journal, achievement quests and game time are skipped.
3. Teleport B's hero to A's hero position after loading (the blob carries B's old position).
4. When saving, move the hero subtree and imported items back to their normal place in the model
   order (after `PlayerJournal`), otherwise the reload restores quests before the hero and fails.

Results:

- Steps 1 to 3: loads into gameplay, hero controllable. Same character (two saves of one
  playthrough) as source and target.
- Step 4, save then fresh start then plain load: hero, items and position intact, no errors beyond
  a normal load of the same save.
- Two different characters (2026-10-18): "Berian" (level 3, 251 gold, 45 items, new game) into
  the world of "Selvar" (level 2, 129 gold, 151 items). After swap, save, fresh start and plain
  load: Berian, level 3, 251 gold, 41 items at Selvar's position; no errors beyond a normal load.
  The 4 missing items are the quest items, keys and readables the New Game+ filter drops.
- Keeping all items (2026-10-18): during export, `NewGamePlusUtils.ShouldItemBeSaved` is
  overridden to keep every item owned by the hero or `HeroStorage`. Berian: 45 of 45 items after
  swap, save and reload.
- Spawn during loading: prefix on `LoadingScreenUI.CleanupAfterLoading` teleports the imported
  hero to host position + 1.5 m along the host's right vector, snapped with
  `AstarPath.active.GetNearest(pos, NNConstraint.Walkable)`. Host at (-95.99, -0.23, 59.27), guest
  spawned at (-94.65, -0.24, 58.62) before the loading screen faded. Setting `Hero.MoveTo` right
  after the blob restore did not stick (position not yet tracked down).
- Orders that did not work: restoring the guest before the host's Gameplay domain (hero needs
  world models); loading the host's Gameplay domain without its hero (quests need the hero).

### Planned save design (owner's decision, 2026-10-18)

- Co-op saves are separate from solo saves. The host's save is the only source of truth: the
  world plus one character file per player, keyed by Steam ID.
- On join the host sends the whole save; the guest loads it and swaps in their own character
  (the transfer above). On save or leave the guest sends their character back to the host.
- One world means no divergence between copies. A player new to a save starts as a copy of the
  host's character (fallback); later maybe a fresh character.
- Co-op uses the host's difficulty.

### Co-op save data (2026-10-19)

- Loading a slot copies **every** zip entry into the domain cache (`LoadSave.LoadSaveSlotToCache`)
  and every save writes **every** cached entry back (`SaveSystem.SaveDomainsAsync`). Extra entries
  therefore travel through saves, autosaves and quicksaves with no hook; unknown names are safe
  (`LargeFilesStorage` creates an empty record on demand). Entries are written with
  `LoadSaveDomainsCache.GetCachedDomainFileWriteStream` + `SaveDomainFileSize`, outside a save.
- End of a save: `SaveInProgressHandle.Dispose` (held for the whole `SaveAsync`).

### Derived (not tested)

- Quest items, keys and readables in a co-op save belong to the same story, so they are kept.
  Edge case: a guest file older than the host's world may carry quest items for finished quests.
- Co-op should use the host's difficulty; Survival and Challenge need special handling.

### Unknowns

- Transfer between saves at very different story points (both test worlds are in the prologue).
- Whether equipped items, skills and talents all survive the transfer.
- Which difficulty effects are computed where once enemies are host-driven.

## Known and expected bugs

What TGCoop players report, the cause where its code shows one, and what our design must do.
"Not built" means the live-sync part does not exist in our work yet.

| Reported bug | Cause in TGCoop 0.5.36 | Our design |
|---|---|---|
| Enemies in different places for the guest; "the mob jumps around while chasing the host" | Guest keeps running its own enemy AI; host sends snapshots every 0.12 s for up to 32 NPCs within 60 m; guest only corrects drift above 1.2 m (2 m in melee) with lerped teleports. Two AIs fight over each enemy. (observed in code) | Not built. One brain per enemy: AI runs on the host only; guest-side NPCs have AI off and interpolate the host's position, rotation and animation state. |
| Enemies mismatched or unsynced | Each player loads their own save, so NPC IDs differ; matched by ID prefix and distance (35 m). (observed in code) | Mostly solved by the single host save: same IDs on both sides at join. NPCs spawned during play need host-assigned IDs. |
| Guest stuck after a dialogue or encounter (Conchur at the asylum, boss fights) | Guest's own story triggers fire locally. (guess, not reproduced) | Not built. Story triggers and scene starts run on the host only; the guest receives results and never enters a blocking story state on its own. |
| No animations in first person | The game never computes a full body pose in first person; TGCoop reconstructs one. (from the author's reply) | Not built. Send movement state (speed, direction, attack, block, cast), animate the remote player on the receiving side with the normal third-person animations. |
| Gate stays closed for the guest; story out of sync | Only `StoryFlags.Set`, quest state and objective state are synced live. (observed in code) | Single host save plus syncing every `ContextualFacts` write. |
| Mod dialogue box covers the game's choices | UI layout. (player report) | Design the spectator dialogue UI so it never overlaps the local one. |
| Only 2 players | Hardcoded. | Character files are per player already; start with 2. |

### Seen in our own tests

- **Guest's tracked quest leaks into the host world** (2026-10-18 join test, old transfer): after
  joining, the guest's HUD tracks "Set a bonfire", a quest from the guest's own world. Not seen with
  the combined load: the guest tracks the host world's "The Great Escape". ("Set a bonfire" turned
  out to be the talent-points hint, not a quest.) The hero's
  `QuestTracker` element travels with the character. The tracked quest must come from the host
  world (or be cleared on import).

- **Received save keeps the host's slot ID** (2026-10-18): saved as `Coop_QuickSave_6.zip` but
  its metadata ID is the host's `QuickSave_6`. At the guest's title scan two slots claim that ID;
  the game renames whichever loads second (alphabetical file order), and `SaveSlot.IsQuickSave`
  is `ID.Contains("QuickSave")`, so a co-op slot can pass for the guest's quicksave. Co-op slots
  need their own IDs.

### Expected problems (derived, not seen yet)

- **Guest crash or quit without saving loses progress**: a guest's character reaches the host's
  co-op save only when the guest saves. Planned with live character sync: regular and
  event-driven pushes, a handshake on clean quit, and a host autosave when a guest leaves.
  A host crash loses pushed characters the host has not saved yet.

- **Enemy targeting with two players**: enemy AI knows only `Hero.Current`; on the host the guest is a puppet. The puppet must become a valid target (attackable, aggro, line of sight) or enemies will ignore the guest.
- **Damage and difficulty**: difficulty multiplies damage dealt and received, stamina and mana use (`Difficulty.DamageDealt/DamageReceived/...`, applied to NPCs via `NpcDifficultyStatModifier`). Hits the guest lands are resolved on the host with the host's difficulty; damage to the guest must use the guest's armour and stats. Decide whether enemies scale with player count.
- **Survival and Challenge difficulty**: save restrictions and a locked hero element; block in co-op at first.
- **Quest items from stale character files**: a guest file older than the host world may carry items for finished quests.
- **Duplicate pickups**: both players looting the same container or drop; host must arbitrate.
- **Time, weather, sleeping, fast travel, wyrdness nights**: world-wide effects one player triggers; host authority needed.
- **Menus pause the game** in single player; in co-op they must not pause the shared world.
- **Death and loading screens**: one player loading a new area or dying must not freeze the other.

## Co-op milestones

| Date | Milestone | Test | Result |
|---|---|---|---|
| 2026-10-19 | Co-op save slots: hosting turns the world into its own co-op slot; on join the host saves it and sends the live world; guests' characters are stored in the host's save (as extra entries the game's domain cache carries through every save) and come back on the next join | `tools/run-suite.ps1`: join, persistence (join, change, save, both quit, rejoin), each with a mutation twin | SUITE: PASS in 470 s (4 cases); guest kept 777 gold across sessions, host 129. |
| 2026-10-18 | Combined load: guest character merged into the host save inside the game's own Gameplay-domain load (host hero dropped at definitions, guest models with fresh item IDs inserted in its place, one RestoreWorld); co-op save made on the guest reloads cleanly without the earlier save-order patch | `tools/run-suite.ps1`: join + reload (15 checks, incl. real end of loading screen) and twin | SUITE: PASS in 270 s; guest HUD tracks the host world's quest. |
| 2026-10-18 | Guest joins host over local UDP (LiteNetLib), receives the host's save (chunked, SHA-256 checked), loads it with its own character next to the host, both stable for 30 s | `tools/run-suite.ps1`: join test + mutation twin (join without character must fail on exactly the 3 character checks), two isolated installs on one PC | SUITE: PASS in 191 s, 10/10 checks per case. |

## Test setup lessons

- The test copy shares `%USERPROFILE%\AppData\LocalLow\Questline\Fall of Avalon` with the real
  install, and the game repacks saves on first launch of this build. Redirect the data folder
  (`CloudService` backing field) and force `SteamNoCloudService` before launching a test copy.
- The game also keeps save scratch data in Unity's `Application.temporaryCachePath`
  (`%LOCALAPPDATA%\Temp\Questline\Fall of Avalon`): `CachedDomains\` (unpacked slot while
  loading) and `SaveFiles.zip` (slot being written). It is shared by the real game and every test
  copy; two copies loading at once corrupted a save. Redirect `IOUtil.TemporaryCachePath`,
  `ZipArchiveUtils.ZipPath` and the `LoadSaveDomainsCache.CachedDomainsDirectoryPath` backing
  field per instance (all static, computed once).
- Two copies run from one install (no single-instance lock in the code), each with its own data
  folder chosen by an environment variable read by the isolation plugin.
- **Isolation must fail closed.** Two copies started at the same moment from one install hit a
  sharing violation on `BepInEx\config\BepInEx.cfg`; the BepInEx preloader threw (written to
  `preloader_<timestamp>.log` in the game folder), Doorstop carried on, and that copy ran
  **unmodded**: `SteamCloudService` and the real save folder, showing a Steam Cloud conflict
  dialog. Real saves were not changed (checked timestamps), but nothing stopped it. Fixes:
  one install per instance (hardlinked game files, own BepInEx folder, plugins folder as a
  junction to the first install's), copies started one after another, and a launch interlock
  that kills a copy unless its isolation plugin reports the expected data folder within 6 s
  (the game initialises its cloud service about 8 s after start). The interlock has its own
  negative test.
- A launch interlock needs failure signals, not a guessed time: the first start after a plugin
  rebuild took longer than 6 s and was killed. Kill on concrete signs instead (BepInEx log not
  rewritten within 5 s, `preloader_*.log` crash file, game log line "Initializing Cloud with
  SteamCloudService"), allow up to 30 s for the confirmation, and wait until every process of the
  install (game and `UnityCrashHandler64`) has exited.
- Never store an install-specific path in a BepInEx config file: a copied config pointed the
  second install at the first install's saves. Derive paths from `Paths.GameRootPath`.
- LiteNetLib 2.1.4 `NetDataWriter.PutBytesWithLength` / `PutArray` store the length as a
  `ushort` and silently truncate arrays over 65,535 bytes. A 104 KB save arrived as a broken zip
  with no error anywhere. Write blobs with an `int` length yourself and send a SHA-256 next to
  anything the receiver writes to disk.
- "In gameplay" must mean the loading screen is gone (`LoadingScreenUI.IsLoading`); the hero model
  exists during loading, so checks and screenshots based on it pass too early.
- A focused test window reads keyboard and mouse: a host screenshot showed the inventory with a
  "Save outfit" dialog open that no bot opened. Test copies should ignore player input.
- Test copies are forced into a 1280x720 window, 30 FPS, `runInBackground`, by patching right
  after `ScreenResolution.Refresh` (where the game applies its display settings).
- Block `AutoBugReporting.SendAutoReport` in test copies: broken experimental loads otherwise send
  reports with saves attached to the developers.
- Automating the title screen: wait for `TitleScreenUI.SpawnTitleScreenUI`; loading earlier
  leaves the title menu drawn over gameplay for the rest of the session. "Press any button" can be
  skipped by setting `TitleScreenUI.s_shownFocusPopup`.
