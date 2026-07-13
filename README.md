# Florida

Follow [FRIDA](https://github.com/frida/frida) upstream, automatically patch and build an anti-detection version of frida-server for Android.

跟随 FRIDA 上游自动修补并构建反检测版本的 frida-server。

**Hint: Don't fork this repository**

## Download

[Latest Release](https://github.com/ret2ldz/Florida/releases/latest)

## What's patched

`build.yml` clones frida main + submodules, applies every `patches/<subproject>/*.patch` in alphabetical order via `git am`, then builds for `android-arm`, `android-arm64`, `android-x86`, `android-x86_64`.

### frida-core patches

| # | Patch | Vector | What it does |
|---|---|---|---|
| 0001 | `string_frida_rpc` | `"frida:rpc"` protocol string | Replaces the literal `"frida:rpc"` RPC magic with a base64-decoded random name, both in outgoing requests and in the inbound match check. |
| 0002 | `frida_agent_so` | `frida-agent-<arch>.so` library name | Renames the agent library template and `frida-agent-arm.so`/`frida-agent-arm64.so` resources to a random UUID prefix so `/proc/self/maps` doesn't show `frida-agent`. |
| 0003 | `symbol_frida_agent_main` | `frida_agent_main` export symbol | Renames the entry-point symbol from `frida_agent_main` to `main` across all host-session files (`agent-container.vala`, `darwin`/`freebsd`/`linux`/`qnx`/`windows`-host-session, tests) and creates `src/anti-anti-frida.py` to rename leftover `frida`/`FRIDA` substrings in the compiled ELF symbols. |
| 0004 | `thread_gum_js_loop` | `gum-js-loop` thread name | Appends a sed-based rename of the `gum-js-loop` thread name (11-char random) to `anti-anti-frida.py`. |
| 0005 | `thread_gmain` | `gmain` thread name | Appends a sed-based rename of the `gmain` thread name (5-char random) to `anti-anti-frida.py`. |
| 0006 | `protocol_unexpected_command` | `Error.PROTOCOL("Unexpected command")` on adb OPEN/CLSE/WRTE | Replaces the throw with a `break;` so droidy doesn't error out on unexpected ADB packets (some custom Android ROMs send extras). |
| 0007 | `update_python_script` | refactor + `FridaScriptEngine`/`GLib-GIO`/`GDBusProxy`/`GumScript` rodata + `gdbus` thread | Refactors `anti-anti-frida.py` with colored logging, extends symbol rename charset, adds rodata reversal of 4 internal Frida/GLib string fingerprints, and adds the `gdbus` thread name patch (5-char random). |
| 0008 | `pool-frida` | `g_set_prgname("frida")` | Replaces `g_set_prgname` argument with `"ggbond"` in `src/frida-glue.c` so `g_get_prgname()` and `prctl(PR_GET_NAME)` don't return `"frida"`. |
| 0009 | `memfd-name-jit-cache` | `memfd_create("frida-agent-<arch>.so", ...)` | Hardcodes the memfd name to `"jit-cache"` in `lib/base/linux.vala` so `readlink /proc/self/fd/N` doesn't reveal the agent. |
| 0010 | `exec-anti-anti-frida.py` | runs the post-build patcher | Modifies `src/embed-agent.py` to invoke `anti-anti-frida.py` against `frida-agent-<flavor>.so` after the embed step. |
| 0011 | `usap-transient` | `Error.NOT_SUPPORTED` on Android-16 USAP spawn | From frida-core PR #1246: tolerate transient `usap*` processes that lack the boot image mappings needed to locate `android.os.Process.setArgV0()`. Swallow the specific failure so spawn proceeds when the main zygote and other USAPs patch successfully. |
| 0012 | `selinux-labels` | `u:object_r:frida_file:s0` / `frida_memfd` SELinux contexts | Renames SELinux labels set by `src/linux/linjector.vala` to `ggbond_file` / `ggbond_memfd`. Apps that call `getfilecon()` on injected files no longer match the `frida_file` type. (Merged from phantom-frida.) |
| 0013 | `exit-monitor-disable` | `interceptor.attach(exit/_exit/abort)` hook | Comments out the exit-monitor hooks in `lib/payload/exit-monitor.vala`. Removes the trampoline gum installs near libc `exit`, which apps detect by calling `exit()` and observing whether a listener fired. (Merged from phantom-frida.) |
| 0014 | `temp-prefix` | `"frida-"` temp-file prefix | Renames the prefix that `Frida.System.make_name()` prepends to random temp file names in `src/system.vala` to `"ggbond-"`. `/data/local/tmp/frida-*` listings no longer match. (Merged from phantom-frida.) |
| 0015 | `pool-spawner-and-string-sweep` | `pool-spawner` thread name + residual `frida\0`/`FRIDA\0` bytes | Extends `anti-anti-frida.py` with: (1) `pool-spawner` thread name patch (12-char random) — the one GLib thread name Florida was missing; (2) `.rodata` sweep for standalone null-terminated `frida\0` → `libgc\0` and `FRIDA\0` → `XBNDL\0`, same-length so binary layout is unchanged. Deliberately skips `Frida\0` (capital F) — that's the JS engine global object (`Frida.version`, etc.) and replacing it crashes `core.js` with `ReferenceError: Frida is not defined`. (Merged from phantom-frida.) |

### frida-gum patches

| # | Patch | Vector | What it does |
|---|---|---|---|
| 0001 | `pool-frida` | `g_set_prgname("frida")` in gum | Same as frida-core 0008 but for `gum/gum.c`. |
| 0002 | `exceptor-disable` | `gum_interceptor_replace(signal)` / `gum_interceptor_replace(sigaction)` | Comments out the two replace calls in `gum/backend-posix/gumexceptor-posix.c`. Without this, app-installed signal handlers don't run because frida's replacement swallows them — the standard detection is "install handler, raise signal, check if handler ran". (Merged from phantom-frida.) |
| 0003 | `perfetto-skip` | `perfetto_hprof_*` thread enumeration crash | In `gum/backend-linux/gumprocess-linux.c`, short-circuits thread enumeration for threads whose name starts with `perfetto_hprof_` (15-char prefix match). Android runtime's perfetto heap profiler spawns these; gum's enumeration segfaults when the thread is mid-`munmap` of its own stack. phantom-frida's version was broken (`goto skip` with no label, typo `perfetto_hprob_`) — fixed here. (Merged from phantom-frida.) |

### Vectors Florida does NOT currently cover

- **Default port 27042.** frida-server listens on 27042 by default; apps `connect()` to it. Changing it source-side breaks the unpatched `pip install frida-tools` client. Fix requires shipping a patched client too — deferred.
- **D-Bus interface names** (`re.frida.HostSession17`, `re.frida.AgentSession17`, ...). Part of the wire protocol; renaming server-side breaks the standard frida client. phantom-frida also disables these. Not visible to apps (only over USB/TCP).
- **Internal GType names** (`FridaServer`/`FridaGadget`/`FridaPortal`/`FridaInject`). phantom-frida renames these via global string replace; risk of breaking the Vala C-symbol linkage, deferred until tested.
- **`/proc/self/status` TracerPid.** No active handling. Apps read TracerPid to detect ptrace attach.
- **Agent load path.** Agent is loaded via `dlopen` from `/data/local/tmp/`, leaves path traces in `dl_iterate_phdr` and `/proc/self/maps`. Could be replaced with `memfd_create` + `mmap` to avoid filesystem footprint.
- **TLS feature values** used by gum interceptor state.
- **QuickJS string constants** in `libfrida-gumjs` (e.g. `<anonymous>`, `JS_Eval`).

## Build workflow

The CI (`.github/workflows/build.yml`) runs every 12 hours on cron, on push to `main`/`ci` when `patches/**` or the workflow itself changes, and on manual dispatch. It:

1. Checks Frida's latest release tag.
2. Creates (or recreates, on push) a release with that tag.
3. Clones frida main, applies all patches in `patches/<subproject>/` via `git am --keep-cr`.
4. Builds for `android-arm`, `android-arm64`, `android-x86`, `android-x86_64`.
5. Packages and uploads 16 gzipped artifacts per release:
   - `florida-server-<ver>-android-<arch>.gz`
   - `florida-inject-<ver>-android-<arch>.gz`
   - `florida-gadget-<ver>-android-<arch>.so.gz`
   - `florida-gumjs-<ver>-android-<arch>.a.gz`

## Recent changes

- **CI fixes:** Added `permissions: contents: write` (was failing with "Resource not accessible by integration" on `create_release`). Bumped `actions/github-script@v3 → v7`, `softprops/action-gh-release@v2.1.0 → v2.2.0` (Node.js 20 deprecation). Bumped checkout/setup-node/setup-java/setup-python to v4/v4/v4/v5. Replaced the 16 deprecated `actions/upload-release-asset@v1.0.2` (Node.js 12, removed mid-2023) with one `gh release upload --clobber` step. Fixed a typo: `florida-inject-...-android-arm-x86_64.gz` → `...-android-x86_64.gz`.
- **Patches 0011–0015 (frida-core) + 0002/0003 (frida-gum):** Merged anti-detection vectors from [phantom-frida](https://github.com/TheQmaks/phantom-frida). See the patch tables above.
- **Patch 0011:** Forward-port of [frida-core PR #1246](https://github.com/frida/frida-core/pull/1246) for Android-16 USAP spawn reliability.

## References

- [https://github.com/hluwa/Patchs](https://github.com/hluwa/Patchs)
- [https://github.com/feicong/strong-frida](https://github.com/feicong/strong-frida)
- [https://github.com/qtfreet00/AntiFrida](https://github.com/qtfreet00/AntiFrida)
- [https://t.zsxq.com/miIunQN](https://t.zsxq.com/miIunQN)
- [https://github.com/darvincisec/DetectFrida](https://github.com/darvincisec/DetectFrida)
- [https://github.com/b-mueller/frida-detection-demo](https://github.com/b-mueller/frida-detection-demo)
- [https://github.com/TheQmaks/phantom-frida](https://github.com/TheQmaks/phantom-frida)

## Thanks

- [@hluwa](https://github.com/hluwa)
- [@feicong](https://github.com/feicong)
- [@r0ysue](https://github.com/r0ysue)
- [@hellodword](https://github.com/hellodword)
- [@qtfreet00](https://github.com/qtfreet00)
- [@TheQmaks](https://github.com/TheQmaks) — phantom-frida, source of merged vectors 0012–0015 / 0002 / 0003
- [@dezige131](https://github.com/dezige131) — frida-core PR #1246, source of patch 0011
