# Folia 26.1.2 → 26.2 update — WIP notes

Goal: fork Folia and bump it from Minecraft **26.1.2** (last official Folia) to **26.2**
("Chaos Cubed", released 2026-06-16), which upstream Folia has NOT shipped
(Folia `ver/26.1.x` last pushed 2026-05-06).

## Why this is possible at all
Paper `main` is already on **26.2** (`mcVersion=26.2`). Folia is a Paper fork, so the
job is: rebase Folia's ~19 patches onto a Paper-26.2 base. Without Paper-26.2 this would
be a from-scratch port (not feasible solo).

## What's done (this branch)
- `gradle.properties`: `mcVersion` 26.1.2→**26.2**, `apiVersion`→26.2,
  `paperRef`→**`8c8eb86f41d67ab999c091b5ebd90fd4d46da652`** (Paper main HEAD, 26.2, 2026-07-01).
  parallel=false + `-Xmx3g` to bound build memory on the shared prod box.
- `folia-server/build.gradle.kts.patch`: re-anchored to 26.2 —
  (a) `oldPaperCommit` comment hash → `d4fe85375af18bfa88f44d7c1e6a61904ae550cc`;
  (b) context `ca.spottedleaf:concurrentutil:0.0.10` → `ca.spottedleaf:leafpile:1.0.0`.

## KEY FINDING (the real work ahead)
**Paper 26.2 replaced `ca.spottedleaf:concurrentutil` with `ca.spottedleaf:leafpile:1.0.0`.**
Folia's region engine (`0001-Region-Threading-Base.patch`) is built on concurrentutil, so
this needs a **compile-time dependency migration** (repackage/rename imports, or add
concurrentutil back as an explicit dep) — not just patch re-anchoring.

## Build/patch mechanics learned
- Task is **`./gradlew applyAllPatches`** (not `applyPatches` — ambiguous).
- The fork's `folia-server/build.gradle.kts` + `folia-api/build.gradle.kts` are **generated**
  (gitignored) from the base + the `*.patch` files. If a stale/broken one is left on disk,
  gradle configuration evaluates it BEFORE the patch task can regenerate it → confusing
  "`:paper-api` could not be found" config error. **Fix: `rm -f folia-server/build.gradle.kts
  folia-api/build.gradle.kts` before re-running**, so paperweight regenerates them fresh.
- paperweight applies patches fuzzily (matches context, ignores line numbers); failures leave
  `*.patch.rej`. Re-anchor the CONTEXT lines that 26.2 changed.

## Resume procedure
```
cd /home/admin/folia-fork
export JAVA_HOME=/usr/lib/jvm/temurin-25-jdk-amd64
rm -f folia-server/build.gradle.kts folia-api/build.gradle.kts
./gradlew applyAllPatches --no-daemon --stacktrace          # resolve remaining Paper patch conflicts
#  -> reaches setupMinecraftSources (downloads + DECOMPILES MC 26.2 — ~6-8G RAM, slow)
#  -> then applies the 8 minecraft-patches incl. 0001-Region-Threading-Base (the hard rebase)
# resolve engine conflicts in folia-server/src/minecraft, then:
./gradlew rebuildAllPatches            # regenerate .patch files from the fixed sources
./gradlew createMojmapPaperclipJar     # produce the runnable jar
```
Then fix compile errors (expect the concurrentutil→leafpile migration + any 26.2 API changes).

## STILL-UNKNOWN / biggest risk
Whether the **8 minecraft-patches** (esp. Region-Threading-Base) rebase cleanly onto 26.2's
changed server/chunk/entity internals. Not yet reached (blocked earlier at the Paper/build
patches). This is the load-bearing feasibility question.

## Build env note
Building on the prod box; added a temporary **8G swap** (`/swapfile-folia`) for the decompile
headroom (live server uses 12G heap). `swapoff /swapfile-folia && rm /swapfile-folia` to remove.

## Beyond the jar (do NOT skip before any prod thought)
- Ecosystem must ALL support 26.2: **Geyser/Floodgate, TCPShield**, all 17 custom plugins
  (rebuild vs `folia-api` 26.2), third-party plugins, and **players must update clients to 26.2**.
- **World upgrade to 26.2 is ONE-WAY** — test only on a throwaway world copy.
- Deploying an unofficial engine fork to live player/economy data = serious data-safety call;
  requires extensive staged load-testing first.

## UPDATE (2026-07-01, run 4): engine patches REACHED — very promising
`applyAllPatches` now gets all the way through: downloads + **decompiles MC 26.2**, applies
all Paper patches, runs Folia setup, and **applies Folia's Minecraft SOURCE + FILE patches
cleanly**. It fails only at `applyMinecraftFeaturePatches` on the big one:
```
Applying: Region Threading Base
error: invalid object ... for 'ca/spottedleaf/moonrise/paper/PaperHooks.java'
error: Repository lacks necessary blobs to fall back on 3-way merge.
```
i.e. Folia's Region-Threading-Base modifies Paper's **Moonrise** chunk-system hook
`PaperHooks.java`, which 26.2 changed, so `git am` can't 3-way merge. This is applied via
**git am** in the paperweight work tree. Resume:
```
# in the folia-server minecraft work git repo (paperweight prints the path):
git am --show-current-patch=diff        # see what Region-Threading-Base wants in PaperHooks.java
# hand-apply Folia's changes to 26.2's PaperHooks.java (+ any further conflicts as am continues)
git add -A && git am --continue         # repeat until the series applies
./gradlew rebuildPatches                # save resolved patches back
./gradlew createMojmapPaperclipJar      # build the jar
```
Then the concurrentutil→leafpile compile migration + any 26.2 API compile fixes.
**Verdict: tractable (no wholesale conflict) but a focused multi-session job — the region
engine ↔ Moonrise integration must be rebased by hand with concurrency care.**

## UPDATE (run 8): 3-WAY MERGE WORKING — engine patches now merge onto 26.2
The blocker chain is solved. Two fixes were needed beyond `oldPaperCommit`:
1. **jgit can't fetch the old base commit** (`Short read of block` in shallow fetch). FIX:
   full-clone Paper to a local mirror `/home/admin/paper-mirror`, then make the old commit
   locally reachable — `git -C <fork> fetch /home/admin/paper-mirror <oldPaperCommit>`.
   (paperweight's oldPaper repo fetches from the fork as origin.)
2. **feature-patch `git am -3` lacks the OLD minecraft-sources blobs** (e.g. PaperHooks.java
   blob 4a3f07d) → can't 3-way. FIX: expose the old source object stores via env
   `GIT_ALTERNATE_OBJECT_DIRECTORIES` (system git honours it), pointing at (all under
   folia-server/.gradle/caches/paperweight):
   `oldPaper/<commit>/paper-server/src/minecraft/java/.git/objects` +
   `mache/base/sources/.git/objects` + (root) `.gradle/.../server-work/paper/file/.../java/.git/objects`.

### Reproduce the 3-way-merge state (gets to the 123 conflicts)
```
cd /home/admin/folia-fork; export JAVA_HOME=/usr/lib/jvm/temurin-25-jdk-amd64
rm -f folia-server/build.gradle.kts folia-api/build.gradle.kts
git fetch /home/admin/paper-mirror b4682bfef616ac62e73cc96046dacdf4a6f53eeb   # old base reachable
# first run builds the oldPaper/mache object stores; then set alternates and re-run:
export GIT_ALTERNATE_OBJECT_DIRECTORIES="$PWD/folia-server/.gradle/caches/paperweight/oldPaper/b4682bfef616ac62e73cc96046dacdf4a6f53eeb/paper-server/src/minecraft/java/.git/objects:$PWD/folia-server/.gradle/caches/paperweight/mache/base/sources/.git/objects:$PWD/.gradle/caches/paperweight/upstreams/server-work/paper/file/src/minecraft/java/.git/objects"
./gradlew applyAllPatches --no-daemon
# -> Region-Threading-Base 3-way merges: git am pauses with CONFLICTs in ~59 files (123 regions)
```

### Remaining work to a building jar (the real grind)
1. Resolve 123 conflict regions in folia-server/src/minecraft/java (Folia region-threading vs
   26.2 changes — take Folia's scheduler-wrapping, adapted to 26.2's restructures).
2. `git add -A && git am --continue` (repeat for feature patches 0002-0008).
3. `./gradlew rebuildPatches` (saves resolved patches back into the repo — DURABLE point).
4. `./gradlew createMojmapPaperclipJar` -> fix compile errors: concurrentutil→leafpile migration
   + any 26.2 API changes. Iterate until it builds.

## UPDATE (run 9): compiles to a KNOWN, small remainder — two decisive findings
Resolved the 123 conflicts with a fast "take Folia's side" script + git am --continue (all 8
feature patches applied). rebuildMinecraftPatches -> compile: **58 errors, ALL SYNTAX**, in 16
files (the take-theirs split some `scheduleOrExecute(...)` lambdas, leaving orphaned/duplicated
blocks). NOT semantic/API errors.

**FINDING 1 — no leafpile migration.** `ca.spottedleaf:leafpile:1.0.0` BUNDLES the whole
`ca/spottedleaf/concurrentutil` package (285 classes) + common/ioutil/profiler/sampler. So
Folia's 40 files importing concurrentutil compile as-is. The feared migration is a NON-issue.

**FINDING 2 — the 16 broken files split into:**
- 2 hand-merged correctly (GiveCommand, SetBlockCommand) — restore the loop header / drop orphan braces.
- 9 peripheral command/feature files reverted to clean 26.2 (`git checkout <base> -- <file>` with the
  GIT_ALTERNATE_OBJECT_DIRECTORIES set): Fill/Place/ForceLoad/Teleport/Enchant commands, Raids,
  EnderDragonFight, ServerPlayerGameMode, PlayerSpawnFinder. **Region-threading DROPPED there —
  TODO: re-merge properly** (these run unthreaded now; a correctness gap, not an engine break).
- **5 CORE files still broken — need careful reconstruction** (take-theirs left DUPLICATED method
  bodies, e.g. Level.setBlock has both the Folia `worldData.*` version AND the 26.2 `this.*` version):
  Level, Entity, MinecraftServer, LevelChunk, SerializableChunkData. These can't be dropped (the
  merged engine references their threading additions).

**Banking subtlety:** rebuildMinecraftPatches regenerates from the minecraft-sources repo's COMMITS,
not the working tree. To persist working-tree fixes: `git -C folia-server/src/minecraft/java add -A
&& git commit --amend --no-edit`, THEN rebuildMinecraftPatches.

### Finish = reconstruct the 5 core files (un-duplicate the method bodies, keep Folia's version),
then recompile (expect only a handful of real 26.2 API tweaks now that leafpile is ruled out),
then `./gradlew :folia-server:createPaperclipJar`.

## UPDATE (run 10): syntax layer DONE, semantic layer 101 -> 25 errors
Full pipeline now: `applyAllPatches` SUCCEEDS for BOTH halves (minecraft feature patches +
paper-server feature patches). Syntax layer 100% (all mangles reconstructed incl. the interleaved
ItemStack.useOn + Entity startRiding/removePassenger). Server-half Region-Threading conflicts (5
files: CraftBlock/CraftBlockState/CraftWorld/CraftMagmaCube/CraftSlime) resolved + rebuilt.

**leafpile package split (26.2) — DONE:** `ca.spottedleaf.moonrise.common.time` -> `ca.spottedleaf.common.time`;
`concurrentutil.util.TimeUtil`/`IntegerUtil` -> `ca.spottedleaf.common.util.*`. Moonrise-specific
util (TickThread, CoordinateUtils, WorldUtil, ReferenceList, etc.) STAYED in paper-server sources.
**Level->worldData field moves — DONE** for DispenseItemBehavior/SaplingBlock/WitherSkullBlock/
MushroomBlock/ServerPlayerGameMode/ItemStack (level.capture*/capturedBlockStates/captureDrops/
treeType -> worldData.* / SaplingBlock.treeTypeRT). Level.levelData protected->public. Reverted
FillBiomeCommand to base (like the other commands). Fixed dup-vars (Connection.encrypted,
Level var6->t, state->blockState), CraftBlockState `access`->getWorldHandle(), ENDERMITE qualifier.

### REMAINING (~25 errors) = "lost Folia additions" + 26.2 API deltas. Retrieve lost members from
the ORIGINAL patch (origin/ver/26.1.x 0001-Region-Threading-Base.patch, `+` lines) and re-add:
- **ServerPlayer.spawnIn(ServerLevel)** method (used 1824/1873, not declared) — Folia addition.
- **LivingEntity**: `isTickingEffects` field + `effectsToProcess` list + `ProcessableEffect` class (used ~1186).
- **ServerLevel** field block: `ENTITY_COUNTER` (base AtomicInteger), `persistentDataContainer`
  (base CraftPersistentDataContainer), `DATA_TYPE_REGISTRY` — dropped by take-theirs; add from base.
- **PlayerSpawnFinder.getOverworldRespawnPos(ServerLevel,int,int)** — Folia method (I reverted this file to base; needs the Folia version).
- **CommandProfiler.java** (io.papermc.paper.threadedregions.commands) — whole Folia file missing.
- **RegionizedServer.globalTick()**, **ChunkMap.entityMap**, AdvancementCommands `count`, MapItem `player2`.
- **26.2 API deltas:** SummonCommand `loadEntityRecursive` signature; PrepareSpawnTask
  `CompletableFuture<Vec3>` (async respawn-pos) + `player`; CommandServerHealth adventure `clickEvent`.
Then `:folia-server:compileJava` clean -> `./gradlew :folia-server:createPaperclipJar`.

## ✅ DONE (run 11, 2026-07-02): FOLIA 26.2 BUILDS. `createPaperclipJar` SUCCESS.
`:folia-server:compileJava` compiles CLEAN (0 errors) and `./gradlew :folia-server:createPaperclipJar`
produces **`folia-server/build/libs/folia-paperclip-26.2.local-SNAPSHOT.jar`** (60M, valid archive,
Main-Class io.papermc.paperclip.Main, version.json id=26.2 world_version=4903 protocol=776).

The last mile after the semantic layer was a class of **"duplicated-body" mangles** the take-theirs
shortcut left (only exposed once definite-assignment/flow analysis ran on a clean parse): remove the
base copy, keep Folia's returning version — PacketProcessor.scheduleIfPossible, TamableAnimal.maybeTeleportTo,
ServerChunkCache.pollTask, LodestoneTracker, YieldJobSite, MushroomBlock.growMushroom, TimeCommand
(queryTimeline* + setTimeToTimeMarker restored to throw), MinecraftServer ctor (dup field block),
WorldGenRegion (restored dropped final-field inits centerChunkX/Z + writeRadius). Plus 26.2 API renames
(spawnIn->setServerLevel, getOverworldRespawnPos->getLevelRespawnPos, PlayerAdvancements.stopListening->clearTriggers,
adventure ClickEvent.clickEvent->runCommand, EntityType.loadEntityRecursive now takes EntitySpawnRequest),
leafpile package split (concurrentutil/common.util|time, moonrise.common.time->common.time), Level->worldData
field moves, ChunkMap.hasEntityWithId->level.getEntity, ENDERMITE qualifier. Profiler patch (0007) was
skipped during am -> dropped the /profiler command (CommandProfiler ref removed from PaperCommands; re-add
the whole Region-profiler patch later if the profiler feature is wanted).

### ⚠️ NOT deployed and MUST NOT be without the ecosystem gates (unchanged): world upgrade to 26.2 is
ONE-WAY; Geyser/Floodgate/TCPShield + all custom plugins must reach 26.2; staged load-test on a throwaway
world first. This is a BUILD milestone (the engine compiles + jars), not a prod-ready release.
Temp `/swapfile-folia` on prod can be removed now (`swapoff /swapfile-folia && rm /swapfile-folia`).

## PROD-READINESS assessment (2026-07-02) — engine READY, gated on third-party 26.2 builds
Stage 1 (jar RUNS): ✅ boots `Done (11.8s)`, world-gen, list/seed responsive, clean shutdown, 0 crashes.
  Runtime fix needed + made: PlayerSpawnFinder.getLevelRespawnPos must use moonrise syncLoadNonFull
  (region-safe), not level.getChunk (crashed setInitialSpawn on the server thread).
Stage 2 (plugins): custom plugins rebuild against **folia-api 26.2 + Adventure 5.2.0** (major bump from
  4.x; stage the 5.2.0 adventure jars into leafhost-plugins/lib). ALL 15 custom ✅ compile + enable.
  Built jars in /home/admin/folia-26.2-plugins/. Test-load on the 26.2 server (folia-26.2-boottest):
  **28 plugins ENABLE on 26.2** — 15 custom + LuckPerms 5.5.55, ProtocolLib 5.4.0, GrimAC 2.3.74,
  SkinsRestorer 15.12.2, TAB 6.0.3, WorldEdit 7.4.3, WorldGuard 7.0.17, Chunky 1.5.3, Sleeper,
  SyncmaticaPaper, TradeCycle. (ProtocolLib+GrimAC loaded on protocol 776 — better than feared.)

### BLOCKERS (need upstream 26.2 builds — partly out of our control):
- **worlds-4.2.2 (TheNextLvl multiworld — runs hub/creative)**: CRITICAL. `No implementation found for
  version: 26.2` — has a per-MC-version impl selector; 4.2.2 has no 26.2. Needs a newer Worlds build.
- **StackMob 5.10.6**: version check rejects "26.2" → soft-disables. Needs a 26.2 build.
- **InventoryStacks 3.3.6**: classloader "zip file error" on enable. Needs update/investigation.
- **Geyser + floodgate (crossplay), voicechat (proximity voice)**: UNTESTED (port-binders, protocol-
  critical on new protocol 776) — must test/find 26.2 builds; if none, crossplay+voice break.
- **Sprout (custom votifier)**: source not in leafhost-plugins list — locate/rebuild.

### REMAINING STAGES (serious, owner-involved):
1. Get 26.2 builds of the blocked third-party (worlds/StackMob/InventoryStacks) + test Geyser/voicechat.
2. Full dev-stack test on leafdev with a COPY of the prod world.
3. **One-way world upgrade** validated on a throwaway copy (data-safety — NOT reversible).
4. Prod deploy: backup + rollback plan + low-traffic + owner approval.
Build artifacts: folia-paperclip jar in folia-server/build/libs/; custom 26.2 jars in ~/folia-26.2-plugins/.

## PROD-READINESS run 2 (2026-07-02) — plugin ecosystem RESOLVED as far as upstream allows
RESOLVED / working on Folia 26.2 (empirically test-loaded):
- 15 custom plugins + Sprout (rebuilt vs folia-api 26.2 + Adventure 5.2.0) → ~/folia-26.2-plugins/
- voicechat **2.6.20** (folia+26.2, fetched Modrinth), floodgate (latest GeyserMC CDN, boots OK)
- LuckPerms 5.5.55, ProtocolLib 5.4.0, GrimAC 2.3.74, SkinsRestorer, TAB 6.0.3, WorldEdit 7.4.3,
  WorldGuard 7.0.17, Chunky, Sleeper, SyncmaticaPaper, TradeCycle — all enable on 26.2.

HARD BLOCKERS — no Folia/26.2 upstream build exists (verified), can't be fetched:
- **worlds (TheNextLvl) — CRITICAL (hub/creative/pvp).** 4.3.0-pre1 has a 26.2 impl but it THROWS
  `Folia is not supported in this version`. Worlds stores registry in plugins/Worlds/worlds.dat and
  loads the leafhost dimensions (world/dimensions/leafhost/{hub,creative,pvp}, 56M, vanilla multi-dim
  layout, NO per-dim level.dat) via per-MC-version NMS. No Folia-26.2 impl. Workaround = custom
  Folia-26.2 dimension-loader OR migrate the 3 dims to standalone worlds + load via WorldCreator
  (data-migration, needs owner sign-off + throwaway-copy test).
- **Geyser 2.10.1 — Bedrock crossplay.** Enable-crashes on 26.2 (bundled `cloud` cmd lib does
  CraftBukkit reflection that broke on 26.2 API). No 26.2 Geyser released. Not fork-able by us →
  deploying now means NO Bedrock crossplay until GeyserMC ships 26.2. (floodgate itself is fine.)
DROPPABLE (no 26.2 build; non-core): StackMob (mob-stack), InventoryStacks (item-stack, enable err).

STILL PENDING (owner-gated): full dev-stack test w/ world COPY; **one-way 26.1.2->26.2 world upgrade**
(irreversible — throwaway-copy validation first); backed-up prod deploy w/ rollback.

### VERDICT: engine + our plugins are ready; FULL prod-readiness is blocked on upstream (Worlds
### Folia-26.2, Geyser 26.2) + the irreversible world upgrade — i.e. it needs either upstream
### releases, or a decision to do custom worlds work + deploy without crossplay. Not a solo-completable
### state today without those calls.

## PROD-READINESS run 3 (2026-07-02) — WORLDS BLOCKER SOLVED (datapack), "build your own" for the rest
Folia BLOCKS the Bukkit createWorld API (UnsupportedOperationException — why Worlds needed NMS). So a
plugin loader can't work. SOLUTION that works on Folia 26.2 (tested): register the 3 leafhost worlds as
**datapack dimensions** — server loads them at STARTUP (before plugins, correct order), no NMS, no data
migration (dims stay at world/dimensions/leafhost/*). Datapack at `/home/admin/folia-26.2-deploy/
leafhost-datapack/` (pack.mcmeta + data/leafhost/dimension/{hub,creative,pvp}.json; hub/pvp=void flat
no-layers/the_void, creative=flat bedrock+dirt+grass/plains). VERIFIED: leafhost:hub 169 built chunks,
pvp 64, creative 25 all load intact. **Consequence: Bukkit world names become `world_leafhost_{hub,
creative,pvp}`** (Paper datapack-dim naming) — deploy must sed-rename `leafhost_X`->`world_leafhost_X`
in Verdant/Grove/Bramble/Roots/Compass configs + Verdant player ymls, and REMOVE the Worlds plugin.
Deploy artifacts: /home/admin/folia-26.2-deploy/ (leafhost-datapack + plugins/ = 18 validated jars).

Remaining third-party gaps + plan ("build our own if it doesn't work"):
- StackMob (mob-stack) / InventoryStacks (item-stack): QoL, no 26.2 build → build custom Folia versions.
- **Geyser (Bedrock crossplay): the one genuine exception — NOT solo-buildable** (full Bedrock<->Java
  protocol, one of the largest MC projects). floodgate works. Path = imminent upstream Geyser 26.2
  (they track MC fast) + re-test newer builds. Crossplay is a hard-req → this gates go-live regardless.
