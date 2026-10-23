# Dove Agent

A Java agent for **Minecraft 1.14.4, 1.15.2 and 1.16.5 on Fabric**, running on **Java 25**, for Dove Client.

One codebase covers the three versions: the patches target Fabric Loader (0.19.5 on all of them)
and DataFixerUpper calls, selected by what the code calls rather than by Minecraft's obfuscated names.

## What it does

| Patch | Where | Effect |
|---|---|---|
| `LazyDataFixerPatch` | Minecraft's `DataFixers` (found by its `DataFixerBuilder.build(Executor)` call) | Minecraft 1.14-1.16 warm DataFixerUpper's rule cache on every start: thousands of tasks on the background threads, 70% of the CPU samples before the title screen on 1.14.4, for rules only needed to open a world saved by an older version. The tasks only fill a cache and drop their results, so they are skipped and each rule is built the first time a fix needs it, as the LazyDFU mod and later Minecraft versions do. **Title screen 6 to 13 s sooner, into a world about 16 s sooner.** |
| `JarFileSystemPatch` | Fabric Loader (`FileSystemUtil`) | Mod jars are opened read-only where Fabric only reads them, skipping a slow Windows writability check (about 0.4 s on 1.21.x; same Fabric Loader here). Fabric's one writer (the runtime mod remapper) keeps its arguments. |

Off switch for both: `fastStartup=false`.

Startup, agent with the changes off and on (Java 25, ZGC, title screen; one round each, 1.16.5 two):

| Version | Off | On |
|---|---|---|
| 1.14.4 | 15.1 s | 9.1 s |
| 1.15.2 | 22.9 s | 9.4 s |
| 1.16.5 | 17.2 / 16.9 s | 12.8 / 11.0 s |

1.14.4 bench, 2 alternating rounds each: title 18.9 -> 9.1 s, into the world 68.1 -> 52.1 s, and the
frame rate unchanged (still 88.9 vs 87.9, turning 87.8 vs 86.2 FPS average; lows within run-to-run
spread). Opening a world saved by 1.14.4 in 1.16.5, which makes DataFixerUpper upgrade it, works the
same with and without the change (same log, joins and plays) and joins sooner (69.0 -> 49.6 s).

## What was checked

**Java 25 needs no fixes on these versions.** They were built for Java 8:

- The jar scan (about 29,000-31,000 classes a version, nested jars included) finds nothing removed
  between Java 21 and 25. What Java 8 had and later Java removed is only used by code these games
  never run: log4j 2.8's email, JMS, database and servlet add-ons, Pack200 in commons-compress, a
  transparent-window helper in JNA, and Netty's self-signed certificate helper.
- Bench sessions (create a world, play, save, quit) and rejoins (open the saved world, play, close
  the window) on all three versions, with FPS Boost on 1.16.5, exit cleanly with the agent. ZGC
  over a whole session: longest pause 0.05 ms (1.14.4) and 0.69 ms (1.16.5), no allocation stalls. One 1.15.2 start crashed natively while creating the window, once, right
  after a test run had been killed; it did not happen again in seven more starts.
- FPS Boost: for 1.14.4 and 1.15.2 none of the six mods exist, so players run Dove alone; 1.16.5
  gets Sodium 0.2, Lithium, FerriteCore and Entity Culling.
- **Frame rate:** no agent change measured as a gain. On 1.14.4 the profile in a world is
  Minecraft's own rendering and chunk work; making that faster is what Sodium does, which the agent
  should not redo.

**Dove itself does not start on Java 8 on these versions** (its Mixin config asks for Java 17), so
they need the launcher's Java 25 setting; Java 17 works too.

## Building

Requires nothing but a JDK to run Gradle; the build downloads a JDK 25 toolchain if one isn't installed.

```bash
./gradlew build
```

The agent is `build/libs/dove-agent.jar`. (The `-slim` jar is the same code without ASM shaded
in; don't use it as the agent.)

## Using

```
-javaagent:/path/to/dove-agent.jar=log=<minecraft dir>/logs/dove-agent.log
--enable-native-access=ALL-UNNAMED
--sun-misc-unsafe-memory-access=allow
```

Agent options (comma-separated after `=`):

| Option  | Default                       | Meaning                                       |
|---------|-------------------------------|-----------------------------------------------|
| `log`   | console only                  | file to also write the agent log to           |
| `cache` | `<java.io.tmpdir>/dove-agent` | directory for cached work                     |
| `debug` | `false`                       | verbose logging (also `-Ddove.debug=true`)    |
| `fastStartup` | `true`                  | `false` turns off the startup changes         |
| `timer` | `false`                       | `true` asks Windows for a 1 ms timer          |
| `renderPriority` | `false`              | `true` raises the render thread's priority    |

## Testing tools

- `./gradlew apiScan --args="--jdk21 <JDK 21 home> [--where java.io.File.delete] [--out report.txt] <jar or folder>..."`
  lists every Java API the given jars (and the jars inside them) use that was removed, is marked for
  removal, or behaves differently on Java 25 than on Java 21, checked against both JDKs' own class files.
- `python tools/mcrun.py --version 1.14.4|1.15.2|1.16.5 --java <home> --dove <jar> --fabric-api <jar> --game-dir <dir> [--agent build/libs/dove-agent.jar[=options]] [--pack <folder>] [--mode bench] [--rejoin] [--jfr <file>]`
  starts the game the way the Dove launcher does (its .minecraft, libraries and Java 25 flags) in a
  separate game folder, and prints the time to the title screen, into the world, and the bench's
  frame rates as one JSON line. `--rejoin` joins the newest world again, plays, and closes the window
  (before 1.20, which has no Quick Play, through the test-only helper mod in `tools/joinworld`).
  For a version the launcher hasn't installed, set `DOVE_MC_DIR` to a folder laid out like `.minecraft`.

## Project layout

```
src/main/java/dev/dove/agent/
├── DoveAgent.java        premain entry point; registers patches
├── AgentOptions.java     log= / cache= / debug= / fastStartup= options
├── transform/            ClassPatch interface, registry (by name or constant-pool marker), transformer
├── patches/              bytecode patches
├── hooks/                runtime code that patched Fabric Loader code (or premain) calls into
├── boot/                 puts the hooks Minecraft's own classes call on the bootstrap class path
└── util/                 console + file logger, constant-pool helpers
src/boot/java/            hooks Minecraft's classes call; shipped under dove-boot/ (see boot/BootClasses)
src/scan/java/            the jar scanner (not part of the agent jar)
tools/mcrun.py            the launch and benchmark harness
```

**Gotcha:** the shaded jar relocates every string literal that looks like `org/objectweb/asm/...`.
To name the game's own ASM classes, build the string at runtime.

**Hooks called from Minecraft's classes go in `src/boot`:** Fabric's class loader hides the
application class path from Minecraft and mods, so those hooks are put on the bootstrap class path
at startup and may only use the JDK.

**These versions are obfuscated:** in a real launch Minecraft's classes are named `net/minecraft/class_1234`
(Fabric's intermediary names), not the Mojang names seen in the dev environment. A patch on a
Minecraft class must target the intermediary name, or better, select classes by what they call
(`markers()`), as `LazyDataFixerPatch` does.
