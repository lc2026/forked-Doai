# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Build commands, verification stages, code style, and repo etiquette live in `AGENTS.md` — read it
first. This file covers commit policy, build details, and architecture that spans multiple files.

> **Never write to `AGENTS.md`. It is read-only.**
> It is maintained upstream — added in commit `ceab821bd4` by a Duranta OAI maintainer and present
> on `origin/develop` — and this fork merges *from* upstream, so any local edit produces a recurring
> merge conflict every time upstream touches the file. Read it, cite it, never modify it. New or
> changed agent guidance goes in **this** file instead.

## Commit messages — this repo forbids the usual AI attribution

The maintainer runs all `git commit`s here; don't commit, push, open PRs, or post PR comments. When
drafting a commit *message*, follow this repo's trailer policy, which overrides any generic
assistant-attribution default.

`doc/code-style-contrib.md`: **AI agents MUST NOT add `Signed-off-by` or `Co-authored-by`
trailers** — only a human can certify the Developer Certificate of Origin. Disclose AI assistance
with `Assisted-by: AGENT_NAME:MODEL_VERSION` instead (e.g. `Assisted-by: Claude:claude-opus-5`).
This is live practice, not aspirational: 123 commits in this history carry `Assisted-by:`. It is a
documented policy rather than a CI gate, though — 222 commits still carry `Co-authored-by:`, so
expect to see violations in the log.

Every commit additionally needs the human author's own `Signed-off-by` (`git commit -s`) *and* a
cryptographic SSH/GPG signature — the DCO check and the verified-commits check are separate CI
gates. See `doc/git-guide.md` for signing setup.

## Build

`doc/BUILD.md` states that running `cmake` directly is the recommended path, so `AGENTS.md` and the
docs agree; `build_oai` is the older wrapper.

**Sanitizers have first-class cmake options — you do not need `build_oai` for them**
`[verified: add_boolean_option calls in CMakeLists.txt]`. `doc/dev_tools/sanitizers.md` documents
only the wrapper flags, which makes them look wrapper-only. They are not:

| `build_oai` flag | cmake option |
|---|---|
| `--sanitize-address` | `-DSANITIZE_ADDRESS=ON` |
| `--sanitize-undefined` | `-DSANITIZE_UNDEFINED=ON` |
| `--sanitize-thread` | `-DSANITIZE_THREAD=ON` |
| `--sanitize-memory` | `-DSANITIZE_MEMORY=ON` (needs clang) |

All default OFF. CMake enforces the MSan-vs-ASan/UBSan incompatibility itself, so a bad combination
fails at configure time rather than at runtime. The `tests` preset already sets
`SANITIZE_ADDRESS=ON`.

**Presets do not build where `AGENTS.md` builds.** Both configure presets set their own `binaryDir`:

| Invocation | Binaries land in |
|---|---|
| `mkdir build && cd build && cmake .. -GNinja` (AGENTS.md) | `build/` |
| `cmake --preset default` | `cmake_targets/ran_build/build/` |
| `cmake --preset tests` (`ENABLE_TESTS=ON`, `SANITIZE_ADDRESS=ON`) | `cmake_targets/ran_build/build_test/` |

So `ctest` must run from whichever directory you actually configured. The historical
`cmake_targets/ran_build/build` path is also why the tutorials' run commands are full of `../../../`.
Build presets: `5gdefault` (NR rfsim targets), `default` (inherits `5gdefault`), `4gdefault`,
`tests` (builds the `tests` target).

**Optional libraries default to OFF and their options are declared in subdirectory CMakeLists, not
the top-level one** — grepping `CMakeLists.txt` alone will not find them, and they use the project's
`add_boolean_option` macro rather than plain `option()`:

```bash
cmake .. -GNinja -DENABLE_TELNETSRV=ON && ninja telnetsrv   # likewise ENABLE_WEBSRV
ccmake ..                                                    # browse every option
```

CI-verified and expected to always build: `telnetsrv`, `enbscope`, `uescope`, `nrscope`.
Known-fragile: `websrv` (npm), `ldpc_aal` (patched DPDK), the scopes (libforms/X).

**`asn1c` is version-sensitive.** Since tag `2023.w22` the project uses the community fork
(`mouse07410/asn1c`), incompatible with the older Eurecom one; both can be installed side by side.
CMake prefers `/opt/asn1c/bin` (`find_program(... HINTS /opt/asn1c/bin)`), and
`-DASN1C_EXEC=/path/to/asn1c` overrides it. Suspect this first when a build breaks immediately after
switching across that tag.

External dependencies come via CPM, cached in `~/.cache/cpm` (`CPM_SOURCE_CACHE`), so a cold build
may fetch from public git repos.

**Design philosophy:** the project deliberately avoids adding build-time options, preferring runtime
config/CLI options — `noS1` and every simulator except the PHY sims were converted from build flags
to runtime flags. Prefer a config option over a new `-D`.

## Running a single test

```bash
ctest -R '^nr_rlc_test$' --output-on-failure   # exact name
ctest -N                                       # list without running
ctest -R '^physim\.nr\.' -L quick_physim       # narrow a physim family
```

Unit tests are *mostly* colocated in a `tests/` or `test/` subdirectory beside the code they cover
(`openair2/LAYER2/nr_rlc/tests/`, `common/config/tests/`, `radio/rfsimulator/tests/`); adding a test
means adding `add_test` to that directory's `CMakeLists.txt`. A handful are not — the repo-root
`tests/` tree holds component testers, and some modules register tests in their own directory. Don't
conclude a test is missing from the colocation pattern alone; get the current list with:

```bash
grep -rl add_test --include=CMakeLists.txt .
```

## One binary, several network functions

`nr-softmodem` is not just "the gNB". Its role is resolved at runtime through `get_node_type()`
(`openair2/GNB_APP/gnb_config.h`), returning an `ngran_node_t` (`common/ngran_types.h`). Use the
`NODE_IS_MONOLITHIC` / `NODE_IS_CU` / `NODE_IS_DU` macros rather than comparing the enum — CU-CP and
CU-UP both count as CU, which is easy to get wrong. In a deployed E1 split the CU-UP runs as its own
binary, `nr-cuup`; note this is a deployment fact, not an enum fact — `get_node_type()` below can
return `ngran_gNB_CUUP` from configuration alone, so don't assume the value implies which
executable you are in.

The role is chosen from configuration by `get_node_type()` (`openair2/GNB_APP/gnb_config.c`), in
this order — read the function, not `doc/F1AP/F1-design.md`, which gives a 3-way simplification that
omits the E1 branch:

1. no `gNBs` list at all → `ngran_gNB` (default)
2. any `MACRLCs.[j].tr_n_preference == "f1"` (northbound) → `ngran_gNB_DU`
3. `gNBs.[0].tr_s_preference == "f1"` (southbound), then by the `E1_INTERFACE` section:
   no E1 section → `ngran_gNB_CU`; `type == "cp"` → `ngran_gNB_CUCP`;
   `type == "up"` → `ngran_gNB_CUUP`; anything else → `ngran_gNB_CU`
4. otherwise → `ngran_gNB` (monolithic)

This matters when tracing any control-plane path: the same source file may run in one process or
three, and whether a call is a direct function call or an inter-node message depends on node type.

| Interface | Connects | Module |
|---|---|---|
| F1AP | CU ↔ DU | `openair2/F1AP` |
| E1AP | CU-CP ↔ CU-UP | `openair2/E1AP` |
| NGAP / NAS | gNB ↔ 5G core (AMF) | `openair3/NGAP`, `openair3/NAS` |
| XnAP | gNB ↔ gNB (handover) | `openair2/XNAP` |
| E2AP / E3AP | RAN ↔ RIC | `openair2/E2AP`, `openair2/E3AP` |

### F1 and E1 are used internally even in a monolithic gNB

Both interfaces are always compiled in, and the *same handlers* run whether the node is integrated
or split — only the transport differs (direct call vs. ASN.1 over SCTP). Dispatch is behind function
pointers, so the direct and F1AP/E1AP implementations sit in parallel files:

| | Callback header | Monolithic impl. | Split impl. | Handler |
|---|---|---|---|---|
| CU → DU | `openair2/RRC/NR/mac_rrc_dl.h` | `mac_rrc_dl_direct.c` | `mac_rrc_dl_f1ap.c` | `NR_MAC_gNB/mac_rrc_dl_handler.c` |
| DU → CU | `NR_MAC_gNB/mac_rrc_ul.h` | `mac_rrc_ul_direct.c` | `mac_rrc_ul_f1ap.c` | ITTI handler in `rrc_gNB.c` |

Note the asymmetry: DL callbacks live with the RRC (the caller, on the CU), UL callbacks live with
the MAC (the caller, on the DU). E1 mirrors it, split across two trees by which side calls
`[verified]`:

- **CP → UP**: `openair2/RRC/NR/cucp_cuup_direct.c` and `cucp_cuup_e1ap.c`
- **UP → CP**: `openair2/LAYER2/nr_pdcp/cuup_cucp_direct.c` and `cuup_cucp_e1ap.c`

**"Direct" means different things in the two directions** `[verified: itti_send_msg_to_task call
counts in each *_direct.c]`:

- **DL direct** (`mac_rrc_dl_direct.c`, 0 ITTI sends) — RRC calls straight into the DU-side handler.
  Synchronous, same thread.
- **UL direct** (`mac_rrc_ul_direct.c`, 15 ITTI sends) — still posts to `TASK_RRC_GNB`. "Direct"
  here removes only the ASN.1 encoding and the socket, **not** the queue.

Don't assume a monolithic build makes both directions synchronous; a UL path that looks like a
function call is still a thread hop.

## Concurrency: three mechanisms, different jobs

- **ITTI** (`common/utils/ocp_itti`, see `itti.md`) — typed message queues that double as thread main
  loops; queues are declared statically and named after their owning thread ("tasks"). Can fold
  sockets and timers into the same loop. This is the control-plane mechanism: RRC, NGAP, F1AP,
  GNB_APP all run as ITTI tasks.
- **Thread pool** (`common/utils/threadPool`) — parallel, short-lived data-plane work (per-slot PHY).
  A lower-level primitive than ITTI, not a replacement.
- **Actor** (`common/utils/actor`) — one thread, one producer/consumer queue. Thread safety comes
  from core affinity or data separation, *not* locking; for concurrency, allocate more actors. The
  nrUE uses these concretely: a Sync Actor for initial sync, DL Actors for DLSCH, a UL Actor for UL.

## PHY ↔ MAC boundary

The L1/L2 split goes through `NR_IF_Module_t` (`openair2/NR_PHY_INTERFACE/NR_IF_Module.h`), a struct
of exactly three function pointers bound at `NR_IF_Module_init()` — all called *by PHY into MAC*
(verified against the header):

```c
void (*NR_UL_indication)(NR_UL_IND_t *UL_INFO);
void (*NR_slot_indication)(const nfapi_nr_slot_indication_scf_t *ind, NR_Sched_Rsp_t *rsp);
void (*NR_PHY_config_req)(NR_PHY_Config_t *config_INFO);
```

- **Uplink**: `NR_UL_indication()` hands MAC everything received in a slot (RACH, CRC, RX data, SRS…).
- **Downlink**: there is **no** separate DL call. PHY calls `NR_slot_indication()`, the scheduler
  runs inside it, and the per-slot DL/UL assignment comes back through the **out-parameter `rsp`**
  (`NR_Sched_Rsp_t`). If you are looking for a `NR_Schedule_response()` function because
  `doc/SW_archi.md` mentions one, it does not exist — that name is stale.
- **Config**: `NR_PHY_config_req()` pushes cell configuration down at setup.

Because it is an indirection layer, the same scheduler in `openair2/LAYER2/NR_MAC_gNB` drives either
the built-in PHY (`openair1`) or an external L1 over FAPI/nFAPI (`nfapi/`) — that is how the 7.2x
fronthaul (`fronthaul/`) and Aerial splits attach. When scheduling misbehaves, first decide whether
the bug is in the scheduler or in the interface marshalling.

On the UE side the equivalent boundary is `nr_ue_dl_indication()` / `nr_ue_ul_indication()` upward
and `nr_ue_scheduled_response()` downward.

## Radio abstraction

All RF goes through one device interface implemented per backend in `radio/`: `rfsimulator`, `USRP`,
`fhi_72` (O-RAN 7.2), plus `iqplayer`, `zmq`, `vrtsim` and vendor SDRs. A change in the common path
affects all of them.

Test against the rfsimulator first because it needs no hardware — but rfsim is the largest single
group in CI, not the majority of it: USRP and fhi72 configurations together outnumber it. An
rfsim-only validation can still break hardware and fronthaul stages. Current split:
`ls ci-scripts/conf_files/ | grep -ci rfsim` versus `usrp` / `fhi72`.

## Configuration

`common/config` implements the config module, with `libconfig` (`.conf`) and `yaml` (`.yaml`)
backends chosen by file extension. Any parameter can be overridden on the command line by its full
dotted path, which is why invocations look like
`--gNBs.[0].NETWORK_INTERFACES.GNB_IPV4_ADDRESS_FOR_NG_AMF 192.168.71.190`. CI-tested reference
configs are in `ci-scripts/conf_files/`.

## Known gotchas

`[verified]` = checked against source in-tree, with the citation. `[doc]` = read in `doc/` and not
independently confirmable without running something.

Citations name a **function or construct**, not a line number — line numbers rot silently on any
edit above them.

- `[verified: RU device-load loop in executables/nr-ue-ru.c]` **`--num-ues N` still opens every
  `RUs` entry**, not just the first N. The loop calling `openair0_device_load()` runs over
  `nrue_ru_count`, never the UE count, so surplus `RUs` entries consume resources unused.
- `[verified: rrc_gNB_process_e1_lost_connection() in openair2/RRC/NR/rrc_gNB_cuup.c]` **A crashed
  CU-UP does *not* require a CU-CP restart** — `doc/E1AP/E1-design.md` says CU-UPs are never
  released, and that is stale. The function clears the E1 assoc ID from every affected UE, then
  `RB_REMOVE`s the CU-UP from `rrc->cuups`, frees it, and decrements `rrc->num_cuups`. Expect
  reconnection to work; if it doesn't, that's a bug, not the documented design.
- `[doc: dev_tools/sanitizers.md]` **ASan builds can die at compile time** with `DEADLYSIGNAL`,
  because OAI runs generated programs *during* the build. Workaround:
  `sudo sysctl vm.mmap_rnd_bits=30` (weakens system ASLR). Not confirmable without building.
- **Submodules** (general git behaviour): `git add -A` / `git commit -a` silently records an
  unintended submodule pointer bump. Stage explicitly with `git add -p`; undo with
  `git restore --staged <path>`.

## Which doc to trust

- **`doc/SW-archi-graph.md`** is the current gNB L1 threading model: `ru_thread()` blocks on RF →
  queue `L1_tx_out` → `L1_tx_thread()` runs the scheduler via `NR_slot_indication` → queue `resp_L1`
  → `L1_rx_thread()` → `L1_rx_out`. TX and RX run in parallel; jobs within each run sequentially.
- **`doc/SW_archi.md`** is broader but partly stale (describes `ocp_rxtx()`, carries `???` markers,
  and flags the `UL_INFO` handling as "not thread safe at all"). Useful for RLC/PDCP/GTP flow;
  prefer the graph doc for L1 threading.
- **`doc/E1AP/E1-design.md`** has one confirmed-stale claim: its "abnormal conditions" section says
  a restarted CU-UP is never restored and CU-UPs are never released from CU-CP structures. Code
  disagrees (see the CU-UP gotcha above). Treat the rest of that doc as current — its E1 message
  flow and config sections check out.
- `doc/MAC/mac-usage.md` decodes the periodic `nrMAC-stats.log` output field by field — the fastest
  way to read a bad-radio run. `doc/RRC/rrc-usage.md` does the same for `nrRRC_stats.log`.
- `doc/README.md` is the index for everything else.
