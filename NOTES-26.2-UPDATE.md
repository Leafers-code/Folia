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
