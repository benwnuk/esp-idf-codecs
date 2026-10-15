# Patched ESP-IDF for ESP32 A2DP codecs

This repository preserves the ESP-IDF snapshot used by the ESP32 A2DP receiver:
<https://github.com/benwnuk/esp-idf-codecs>.

It is a community-maintained, modified SDK, **not an official Espressif release**.
It targets the original ESP32 with Bluetooth Classic. Other chips, examples, and
optional features inherited from ESP-IDF are not certified by this project.

## Origins and reproducibility

| Layer | Source |
| --- | --- |
| Official comparison baseline | [Espressif ESP-IDF v5.5.2](https://github.com/espressif/esp-idf/tree/v5.5.2), commit `30aaf64524299d3bde422ca9a2848090d1bc5d0f` |
| Imported multicodec SDK | [WillyBilly06/ESP32-A2DP-SINK-WITH-CODECS-UPDATED](https://github.com/WillyBilly06/ESP32-A2DP-SINK-WITH-CODECS-UPDATED), commit [`f4eb12517438a486af9e33987114c4823af40aca`](https://github.com/WillyBilly06/ESP32-A2DP-SINK-WITH-CODECS-UPDATED/commit/f4eb12517438a486af9e33987114c4823af40aca), directory `esp-idf/` |
| Earlier codec integration credited by the reference project | [cfint/ESP32-A2DP](https://github.com/cfint/ESP32-A2DP) and [WillyBilly06/esp32-a2dp-sink-with-LDAC-APTX-AAC](https://github.com/WillyBilly06/esp32-a2dp-sink-with-LDAC-APTX-AAC) |
| Receiver-specific changes | The three source changes described below, carried from the local ESP32 A2DP receiver workspace |

The initial commit is a source snapshot, not an import of Espressif's or the
reference application's Git history. The reference checkout was shallow; the
comparison baseline identifies observable differences, not a claim that every
change originated in a particular upstream commit.

Dependencies that were already flattened into the reference SDK remain vendored
as ordinary files, including required Espressif controller/radio libraries.
There are **no nested Git submodules** to initialize. The original submodule
declarations are retained as [.gitmodules.upstream](.gitmodules.upstream) for
provenance only; they are not active configuration and do not prove the exact
revision of every vendored dependency.

Python bytecode/cache directories, macOS metadata, and the inherited root GitHub,
GitLab, and Read the Docs automation were excluded. That automation assumed
Espressif infrastructure or upstream repository permissions. Source examples,
tests, third-party notices, and required binary libraries were retained.
Executable modes for files present in official v5.5.2 follow that release;
other imported files retain the reference commit's modes.

## Getting the SDK

For the receiver, use its pinned submodule rather than choosing this repository's
latest branch:

```sh
git submodule update --init --recursive
```

For standalone SDK use:

```sh
git clone --branch v5.5.2-codecs.1 https://github.com/benwnuk/esp-idf-codecs.git
cd esp-idf-codecs
./install.sh esp32
. ./export.sh
```

On Windows, use `install.bat esp32` followed by `export.bat` in Command Prompt.
Install the host prerequisites from the
[ESP-IDF 5.5.2 setup guide](https://docs.espressif.com/projects/esp-idf/en/v5.5.2/esp32/get-started/index.html).
Installation downloads host tools and Python packages; a source clone is not an
offline toolchain bundle. `IDF_TOOLS_PATH` can select a shared tool installation;
otherwise ESP-IDF uses its normal per-user location.

The initial SDK release is tagged `v5.5.2-codecs.1`. The SDK's numeric version
remains 5.5.2 for component compatibility; Git identifies the patched release.
Pin application submodules to a tested commit. Do not move published tags.

## Differences from stock ESP-IDF 5.5.2

The following describes the imported implementation, not a promise that every
inherited change is required or independently tested.

### Codec integration

- Optional decoder configuration and build sources for AAC, the aptX family,
  LDAC, and Opus, alongside SBC. Legacy LC3plus configuration/source references
  also remain; they are not evidence of working LC3plus or LE Audio support.
- Codec capability parsing, vendor codec selection, decoder interfaces, and
  PCM delivery integrated into Bluedroid A2DP.
- Public `esp_a2dp_api.h` codec capability extensions. Stock ESP-IDF is not a
  drop-in replacement for applications using these extensions.
- Bundled Helix AAC, libfreeaptx, libldac-dec, and Opus sources in
  [components/bt/host/bluedroid/external](components/bt/host/bluedroid/external).
- Codec-dependent internal-RAM scratch buffers and compressed packet queue
  limits, decoder reset/lifetime handling, and memory-pressure queue relief.
  Some inherited comments describe older sizes; the actual definitions and
  codec-specific functions determine the current values.

### Bluetooth transport and resource handling

Inherited changes also affect the BTA A2DP/AVRCP state machines, AVDTP, AVCTP,
L2CAP, HCI, and BTU paths. They include connection handling, diagnostics, buffer
lifetime/resource handling, and an HCI reassembly allocation retry that asks
A2DP to release queued buffers before dropping a packet on continued failure.

The Bluetooth OS abstraction adds optional PSRAM-backed thread creation and
null-safe semaphore cleanup. The receiver itself disables PSRAM; retaining that
optional path does not mean it was tested in this application.

### Changes outside Bluetooth

- `components/esp_system/port/cpu_start.c`: reorders the ESP32 secondary CPU
  boot-address, reset/clock, cache, and unstall operations.
- `components/esp_wifi/CMakeLists.txt`, `Kconfig`, and added
  `src/esp_wifi_ecast.c` / `include/esp_wifi_ecast.h`: retain an optional
  EspCastBR low-latency Wi-Fi profile. This is inherited functionality, not
  Bluetooth LE Auracast, and is not needed by the receiver's Classic A2DP path.

### Receiver-specific fixes retained in this snapshot

| File | Change and rationale |
| --- | --- |
| [bta_av_act.c](components/bt/host/bluedroid/bta/av/bta_av_act.c) | Initiate the outgoing AVRCP controller connection after SDP even when an idle acceptor listener exists. This removes the reference project's listener-based suppression and largely restores stock v5.5.2 behavior. Check the returned handle before indexing the control block; log discovery/open results. |
| [sdp_discovery.c](components/bt/host/bluedroid/stack/sdp/sdp_discovery.c) | Use a 512-byte buffer for the service-search-attribute initial/continuation request, instead of the large default allocation, to avoid a previously observed 4,112-byte contiguous-allocation failure. Add explicit allocation diagnostics with internal free/largest-block sizes. Incoming response capacity and other SDP allocation paths are unchanged. |
| [btc_a2dp_sink.c](components/bt/host/bluedroid/btc/profile/std/a2dp/btc_a2dp_sink.c) | Fix the unhandled-event warning to use `p_buf->event` instead of undefined `sig`, including when that warning is compiled in. |

The SDP request allocation is tied to the current filter/continuation limits.
Recheck its maximum serialized size if those limits or packet construction
change. The AVRCP change supersedes a workaround intended for Realtek-like
peers; include those peers in future reconnect regression testing.

### Comparison inventory

A content comparison against files in official v5.5.2 found changes in the
following 48 files. This list excludes new codec/vendor dependency files,
metadata cleanup, file modes, and this repository's documentation. It is an
inventory, not an attribution or validation of every hunk.

```text
components/bt/CMakeLists.txt
components/bt/common/osi/semaphore.c
components/bt/common/osi/thread.c
components/bt/common/osi/include/osi/thread.h
components/bt/host/bluedroid/Kconfig.in
components/bt/host/bluedroid/api/include/api/esp_a2dp_api.h
components/bt/host/bluedroid/bta/av/bta_av_aact.c
components/bt/host/bluedroid/bta/av/bta_av_act.c
components/bt/host/bluedroid/bta/av/bta_av_main.c
components/bt/host/bluedroid/bta/av/bta_av_ssm.c
components/bt/host/bluedroid/bta/av/include/bta_av_int.h
components/bt/host/bluedroid/bta/include/bta/bta_av_api.h
components/bt/host/bluedroid/bta/include/bta/bta_av_co.h
components/bt/host/bluedroid/btc/profile/std/a2dp/bta_av_co.c
components/bt/host/bluedroid/btc/profile/std/a2dp/btc_a2dp.c
components/bt/host/bluedroid/btc/profile/std/a2dp/btc_a2dp_sink.c
components/bt/host/bluedroid/btc/profile/std/a2dp/btc_a2dp_sink_ext_coedc.c
components/bt/host/bluedroid/btc/profile/std/a2dp/btc_av.c
components/bt/host/bluedroid/btc/profile/std/a2dp/include/btc_av_co.h
components/bt/host/bluedroid/btc/profile/std/include/btc_a2dp_sink.h
components/bt/host/bluedroid/btc/profile/std/include/btc_av.h
components/bt/host/bluedroid/btc/profile/std/include/btc_av_api.h
components/bt/host/bluedroid/common/include/common/bluedroid_user_config.h
components/bt/host/bluedroid/common/include/common/bt_target.h
components/bt/host/bluedroid/hci/hci_hal_h4.c
components/bt/host/bluedroid/hci/packet_fragmenter.c
components/bt/host/bluedroid/stack/a2dp/a2d_api.c
components/bt/host/bluedroid/stack/a2dp/a2d_sbc.c
components/bt/host/bluedroid/stack/avct/avct_l2c.c
components/bt/host/bluedroid/stack/avct/avct_lcb_act.c
components/bt/host/bluedroid/stack/avdt/avdt_api.c
components/bt/host/bluedroid/stack/avdt/avdt_l2c.c
components/bt/host/bluedroid/stack/avdt/avdt_msg.c
components/bt/host/bluedroid/stack/avdt/avdt_scb_act.c
components/bt/host/bluedroid/stack/avdt/include/avdt_int.h
components/bt/host/bluedroid/stack/btu/btu_task.c
components/bt/host/bluedroid/stack/include/stack/a2d_api.h
components/bt/host/bluedroid/stack/include/stack/a2d_sbc.h
components/bt/host/bluedroid/stack/include/stack/btu.h
components/bt/host/bluedroid/stack/l2cap/l2c_csm.c
components/bt/host/bluedroid/stack/l2cap/l2c_link.c
components/bt/host/bluedroid/stack/l2cap/l2c_main.c
components/bt/host/bluedroid/stack/l2cap/l2c_utils.c
components/bt/host/bluedroid/stack/l2cap/include/l2c_int.h
components/bt/host/bluedroid/stack/sdp/sdp_discovery.c
components/esp_system/port/cpu_start.c
components/esp_wifi/CMakeLists.txt
components/esp_wifi/Kconfig
```

## Credits and licenses

All original copyrights and notices remain with their respective owners.
Credit includes Espressif Systems and its contributors; the reference-project
maintainers and earlier codec-integration authors linked above; Sony and the
libldac-dec contributors; Aurelien Jacobs, Pali Rohar, Hunter Wardlaw and
libfreeaptx contributors; RealNetworks/Helix contributors; Xiph.Org and Opus
contributors; and the other vendored dependency authors.

**This repository is not uniformly MIT-licensed.** The reference application's
MIT README label does not relicense its SDK or codec dependencies.

| Material | Notices shipped in this snapshot |
| --- | --- |
| ESP-IDF framework | Root [LICENSE](LICENSE) is Apache-2.0; individual files and bundled components can have other terms. |
| aptX decoder implementation | [libfreeaptx/main/COPYING](components/bt/host/bluedroid/external/libfreeaptx/main/COPYING) and `freeaptx.c` identify LGPL-2.1-or-later. The outer [libfreeaptx/LICENSE](components/bt/host/bluedroid/external/libfreeaptx/LICENSE) describes wrapper code as public domain/CC0, not the entire decoder. |
| Helix AAC | Source headers, including [aacdec.c](components/bt/host/bluedroid/external/libaac-lc/libhelix_aac/aacdec.c), reference RealNetworks RPSL/RCSL terms. No standalone full license text was supplied in that directory. |
| LDAC decoder | [libldac-dec](components/bt/host/bluedroid/external/libldac-dec) retains supplied headers, including Sony copyright notices. No standalone license file was supplied in that directory; this import does not establish missing permissions or a uniform license for it. |
| Opus | [COPYING](components/bt/host/bluedroid/external/opus/COPYING) and [LICENSE_PLEASE_READ.txt](components/bt/host/bluedroid/external/opus/LICENSE_PLEASE_READ.txt), plus file-specific notices. |
| Other sources and binary libraries | Component-local notices, headers, and license files, including Espressif binary-library terms. These libraries are dependencies, not build artifacts to delete. |

Preservation of the reference snapshot is not a legal clearance or a grant of
codec patent/trademark rights. Review the actual licenses, missing license-text
provenance, and applicable patent requirements before redistribution or commercial
use, especially before distributing statically linked firmware. A Git submodule
does not remove those obligations. Do not invent a permissive replacement license
for unclear third-party material.

## Validation and maintenance

The receiver project previously exercised SBC, AAC, and standard aptX, and
Samsung A71 AVRCP metadata/absolute volume. aptX-HD, aptX-LL, LDAC, Opus, optional
PSRAM paths, Wi-Fi additions, and other targets are not certified by this import.
A successful build does not prove codec negotiation or real-time playback.

This extraction intentionally preserves the known receiver SDK behavior rather
than removing inherited transport changes. For an SDK update:

1. Keep source provenance and notices, and document the changed files.
2. Build the receiver against the new SDK commit from a clean build directory.
3. Test pairing/reconnection, AVRCP metadata/volume, codec negotiation, and
   playback/resource headroom on representative peers.
4. Commit and publish the SDK first, then update the receiver's submodule pin.

Do not use `git submodule update --remote` as a substitute for selecting and
testing a specific SDK commit.
