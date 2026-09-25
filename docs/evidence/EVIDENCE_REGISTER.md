# Evidence Register

**Status:** NORMATIVE for what AutoDiag may execute.
**Source pin:** `guns96x/ghidracarista` @ `f4b30e5246d36c77a186f421a6c503392eacbfb2` (2026-09-25). Every path below is relative to that repository at that commit.
**Rule:** Gemini implements automotive behavior **only** from rows with status `ACCEPTED` or `OFFICIAL_PROTOCOL`. Rows with `CONFLICT`, `REJECTED` or `BLOCKED_EVIDENCE` are never turned into executable code paths or runtime-pack entries. No new binary reverse engineering is performed for this project; new evidence enters only through a new commit in `ghidracarista`, followed by an update of this register (ADR-0002).

## 1. Status vocabulary

| Status | Meaning for AutoDiag |
|---|---|
| `ACCEPTED` | Committed evidence with a concrete source (function and line or address) and no contradicting committed evidence. May back an `EXACT` runtime-pack field. |
| `OFFICIAL_PROTOCOL` | Behavior defined by a public standard or vendor datasheet (ISO 15765-2, ISO 14229-1, SAE J1979, ELM327 datasheet, FTDI AN232B-05, USB CDC 1.2). May back **generic protocol-engine behavior** and **read-only standardized identifiers**. It never backs a write, an OEM routine, or an applicability decision on its own. |
| `CONFLICT` | Two committed sources disagree. Non-executable until one source is retracted in `ghidracarista`. |
| `REJECTED` | Refuted by an audit (Round 2–4) or proven synthetic. Never used. |
| `BLOCKED_EVIDENCE` | Required for a capability, but not present in committed evidence. The row states exactly what evidence would unblock it. |

## 2. Transport

| ID | Fact | Source | Status |
|---|---|---|---|
| EV-TR-01 | SPP UUID `00001101-0000-1000-8000-00805F9B34FB` | `research/protocols/transport.md` §2 (classes.dex string table, `AndroidBluetooth2Connection`) | ACCEPTED |
| EV-TR-02 | RFCOMM connect timeout 10 000 ms; connect retries: 2 | `research/state_machines/connection_lifecycle.md` §3 (`ConnectionManager`, lines 1347172–1347445) | ACCEPTED |
| EV-TR-03 | Adapter responses are CR-terminated; `>` prompt signals readiness | `research/protocols/transport.md` §4 | ACCEPTED |
| EV-TR-04 | Command response timeout 5 000 ms; extended diagnostic service timeout 10 000 ms | `research/protocols/transport.md` §4 | ACCEPTED |
| EV-TR-05 | BLE: serialized GATT command queue; `requestMtu(512)`; CCCD `00002902-0000-1000-8000-00805f9b34fb`; write chunks limited by negotiated MTU | `research/protocols/transport.md` §3 (`Bluetooth4Gatt`) | ACCEPTED |
| EV-TR-06 | BLE profile `FFF0` service, `FFF1` notify, `FFF2` write | `research/protocols/transport.md` §3 item 1 | ACCEPTED |
| EV-TR-07 | BLE profiles for OBDLink CX and Kiwi 3 exist as classes | `research/protocols/transport.md` §3 items 2–3 | BLOCKED_EVIDENCE — service/characteristic UUIDs are not committed. Needed: UUID constants from `Bluetooth4Profile$ObdLinkCx` and `Bluetooth4Profile$Kiwi3` committed with dex references. |
| EV-TR-08 | Device-name keyword list (`OBD`, `Carista`, `vLinker`, `Vgate`, `Viecar`, `OBDLink`) | `research/protocols/transport.md` §2 | ACCEPTED as a **sorting hint only**; it never filters devices out and never implies capabilities |
| EV-TR-09 | USB serial: FTDI vendor requests (reset, set baud rate, set data, set latency timer), CDC-ACM `SET_LINE_CODING`/`SET_CONTROL_LINE_STATE` | FTDI AN232B-05 / USB CDC 1.2 | OFFICIAL_PROTOCOL |
| EV-TR-10 | K-Line (ISO 9141 / ISO 14230 physical) support in the reference application | `connection_lifecycle.md` mentions "Fallback to ISO 9141 / KWP K-Line" without source lines | BLOCKED_EVIDENCE — K-Line is out of v1 scope. Needed: committed K-Line init/request evidence (functions + lines). |

## 3. Adapter (ELM327 / STN / vLinker)

| ID | Fact | Source | Status |
|---|---|---|---|
| EV-AD-01 | Init sequence `ATZ`, `ATE0`, `ATL0`, `ATS0`, `ATH1`, `ATAT1` | `research/protocols/elm327.md` §2 (`ElmSimulator::processAtCommand`, lines 1009425–1011910) | ACCEPTED |
| EV-AD-02 | `ATCAF0` / `ATCAF1` used; clones may answer `?` to `ATCAF0` | `elm327.md` §2 row 7 and §4 | ACCEPTED |
| EV-AD-03 | `ATSH`, `ATCF`, `ATCM`, `ATCP`, `ATCEA`, `ATSP6`, `ATSP7`, `ATSPB` | `elm327.md` §3 (lines 1009497–1009514) | ACCEPTED |
| EV-AD-04 | Adapter identity strings: `Carista EVO`, `vLinker ` (with trailing space), OBDLink marker `1150` | `elm327.md` §4 (`Elm::setAdapterType`) | ACCEPTED as identity evidence; product-name → capability mapping is REJECTED (capabilities are probed, §5 of `TRANSPORT_ADAPTER_CONTRACT.md`) |
| EV-AD-05 | STN commands `STDI`, `STP`, `STPX`, `STCSM`, `STCMM` exist | `research/protocols/stn.md` §2 (`ElmSimulator::processStCommand`, lines 1013023–1013400) | ACCEPTED |
| EV-AD-06 | `STP 33` selects TP2.0 on STN | `research/molecular/transport_state_machines/elm327_stn_state_machine.json` (no source line) | BLOCKED_EVIDENCE — needed: source function/line for the STN protocol number used for TP2.0. |
| EV-AD-07 | Adapter error tokens `NO DATA`, `CAN ERROR`, `BUS BUSY`, `FB ERROR`, `BUFFER FULL`, `ERR<nn>`, `?` | ELM327 datasheet; also `elm327_stn_state_machine.json` | OFFICIAL_PROTOCOL |
| EV-AD-08 | `AT RV` returns battery voltage `<d>.<d>V` | ELM327 datasheet | OFFICIAL_PROTOCOL |
| EV-AD-09 | `ATST` response-timeout units of 4 ms; `ATLP` low-power | ELM327 datasheet | OFFICIAL_PROTOCOL |
| EV-AD-10 | Per-AT-command timeout 2 000 ms, 3 retries; bus-init timeout 5 000 ms; error-recovery backoff 3 000 ms, max 3 attempts, then `ATLP` and disconnect | `connection_lifecycle.md` §3 | ACCEPTED |
| EV-AD-11 | Protocol probe order `ATSP6` → `ATSP7` → `ATSPB` | `connection_lifecycle.md` §2–3 | ACCEPTED |
| EV-AD-12 | `ATSPB` custom CAN parameters (`ATPB` value) for TP2.0 | not committed | BLOCKED_EVIDENCE — needed: the exact `ATPB xx yy` (or equivalent) string Carista sends before `ATSPB`, with source line. |

## 4. CAN / ISO-TP

| ID | Fact | Source | Status |
|---|---|---|---|
| EV-CAN-01 | UDS 11-bit response ID rule: `TX >= 0x795 → RX = TX + 0x08`, else `RX = TX + 0x6A` | `research/protocols/can.md` §2 (`VagUdsEcu::VagUdsEcu`, lines 584009–584012) | ACCEPTED (used only as a cross-check of explicit pack addresses) |
| EV-CAN-02 | 29-bit response ID rule: `RX = TX + 0x00020000` | `can.md` §2 (line 583921) | ACCEPTED (cross-check only) |
| EV-CAN-03 | UDS address table (VAG ID → TX/RX) in `research/protocols/vag_addressing.md` §2, UDS columns | `VagUdsEcu::initialize()` line 583306 | ACCEPTED for the UDS TX/RX columns of all 18 rows; every row satisfies EV-CAN-01 |
| EV-ISO-01 | ISO-TP SF/FF/CF/FC structure, FS values 0/1/2, 12-bit FF length | ISO 15765-2 | OFFICIAL_PROTOCOL |
| EV-ISO-02 | Tester flow control `30 00 00` (BS 0, STmin 0) | `research/protocols/isotp.md` §2 and `iso_tp_state_machine.json` (no source line) | BLOCKED_EVIDENCE for host-side ISO-TP; not needed when the adapter performs flow control (offload mode). Needed: source line for the FC frame bytes and padding byte used by Carista's host-side ISO-TP. |
| EV-ISO-03 | ISO-TP transmit padding byte for VAG ECUs | not committed | BLOCKED_EVIDENCE — host-side ISO-TP transmit is non-executable until committed. |
| EV-ISO-04 | N_As/N_Ar/N_Bs/N_Cr = 1 000 ms; max FC.WAIT = 10 | `iso_tp_state_machine.json` (no source line) | OFFICIAL_PROTOCOL values (ISO 15765-2 defaults) are used; the JSON is treated as corroboration only |

## 5. UDS

| ID | Fact | Source | Status |
|---|---|---|---|
| EV-UDS-01 | Request/response framing for 0x10, 0x11, 0x14, 0x19, 0x22, 0x27, 0x2E, 0x31, 0x3E; negative response `7F <SID> <NRC>`; positive SID = SID + 0x40 | ISO 14229-1 | OFFICIAL_PROTOCOL |
| EV-UDS-02 | Extended session `10 03` | `research/db/diagnostic_evidence.json` `vag_uds_session_extended` (`VagUdsEcu::enterExtendedSession` 0x583306) | ACCEPTED |
| EV-UDS-03 | Tester present `3E 80` (suppress positive response) | `diagnostic_evidence.json` `vag_uds_tester_present_suppressed` (`TesterPresentCommand` 0x166630c) | ACCEPTED |
| EV-UDS-04 | Tester-present period 2 000 ms; S3 (session timeout) 5 000 ms | `research/protocols/sessions.md` §2 (no line) + ISO 14229-2 S3server = 5 000 ms | OFFICIAL_PROTOCOL (S3); 2 000 ms period ACCEPTED as policy value below S3 |
| EV-UDS-05 | NRC 0x78 extends wait to P2* (default 5 000 ms) | ISO 14229-2; `research/protocols/uds.md` §3 | OFFICIAL_PROTOCOL |
| EV-UDS-06 | Default P2 = 50 ms | ISO 14229-2; `connection_lifecycle.md` §3 | OFFICIAL_PROTOCOL; over ELM adapters the effective wait is the adapter timeout (see `TRANSPORT_ADAPTER_CONTRACT.md` §6) |
| EV-UDS-07 | DTC read `19 02 8D`; record = 3-byte DTC + 1 status byte after `59 02 <availabilityMask>` | `diagnostic_evidence.json` `vag_uds_read_dtc_8d`; `research/protocols/dtc.md` §2 | ACCEPTED |
| EV-UDS-08 | Clear DTC `14 FF FF FF` → `54` | `diagnostic_evidence.json` `vag_uds_clear_dtc` | ACCEPTED |
| EV-UDS-09 | Identification DIDs: `F18C` serial, `F189` SW version, `F197` component name, `F19E` ASAM/ODX ID, `F1A2` ASAM revision, `F1A5` workshop code, `0608` slave/submodule IDs, `539B` diagnostic filter status | `diagnostic_evidence.json` (`vag_uds_ecu_*`, `vag_uds_slave_submodule_ids`, `vag_uds_diag_filter_status`) | ACCEPTED (read-only) |
| EV-UDS-10 | `F187` spare part number | `research/features/applicability_rules.json` `RULE_PART_NO_WHITELIST` (line 1353215) + ISO 14229-1 Annex C | ACCEPTED (read-only) |
| EV-UDS-11 | `F190` VIN (engine `0x7E0` or gateway `0x710`); OBD `09 02` broadcast to `0x7DF` | `research/molecular/ecu_identification_rules.json` stage 2 + ISO 14229-1 / SAE J1979 | OFFICIAL_PROTOCOL (read-only) |
| EV-UDS-12 | ASAM identifier DID used for applicability: `F19E` vs `F15B` | `diagnostic_evidence.json` says `F19E`; `applicability_rules.json` `RULE_ASAM_ODX_MATCH` says `22 F1 5B` | CONFLICT — ASAM-based applicability is non-executable until resolved. `F19E` remains ACCEPTED for display (EV-UDS-09). |
| EV-UDS-13 | Hardware version DID | not committed with a source | BLOCKED_EVIDENCE — `EcuIdentity.hardwareVersion` stays `null` on UDS. Needed: DID + source line. |
| EV-UDS-14 | Security access message shapes `27 <odd>` / `67 <odd> <seed>` / `27 <even> <key>` | ISO 14229-1; `research/protocols/security_architecture.md` §2 | OFFICIAL_PROTOCOL for message shape only; **no key algorithms exist in evidence and none may be written** |
| EV-UDS-15 | SFD commands `31 01 C0 08`, `31 01 C0 <token>`, `31 01 C0 12 01`, `31 01 06 A9 00` | `security_architecture.md` §3 | ACCEPTED for **detection only**; sending `31 01 C0 …` unlock/confirm requests is forbidden (spec §33) |

## 6. VAG discovery and identity

| ID | Fact | Source | Status |
|---|---|---|---|
| EV-DSC-01 | UDS gateway list: `22 04 A1` to gateway TX `0x710` / RX `0x77A`; positive `62 04 A1`; payload after DID echo is a multiple of 4; byte 0 of each record = VAG ECU address | `research/protocols/ecu_discovery.md` §2 (`GetVagUdsEcuListCommand`, lines 107134/107148/107281) | ACCEPTED |
| EV-DSC-02 | Meaning of record bytes 1–3 | `ecu_discovery.md` (byte 1 flags, byte 2 installed / bit 2 active, byte 3 extension) vs `research/molecular/DIAGNOSTIC_OPERATIONS_INDEX.json` (byte 1 status mask, byte 2 bus type, byte 3 protocol ID); class names also differ (`GetVagUdsEcuListCommand` vs `GetVagUdsInstalledEcusCommand`) | CONFLICT — AutoDiag stores bytes 1–3 raw and derives nothing from them. Presence is decided by the identity probe (`STATE_MACHINES.md` §7). |
| EV-DSC-03 | TP2.0 gateway list `1A 9F` → `5A 9F` | `diagnostic_evidence.json` `vag_tp20_gateway_install_list` | ACCEPTED as request/response prefix; record layout BLOCKED_EVIDENCE (needed: `GetVagCanEcuListCommand::processPayloads` parsing rules with lines). |
| EV-DSC-04 | NRC 0x31 on `22 04 A1` falls back to KWP `1A 9B` | `DIAGNOSTIC_OPERATIONS_INDEX.json` (no line) | BLOCKED_EVIDENCE |

## 7. TP2.0 and KWP2000

| ID | Fact | Source | Status |
|---|---|---|---|
| EV-TP-01 | Data-packet opcodes: `0x10` data without ACK request, `0x20` data with ACK request, `0xB0` ACK, `0xA0` keepalive/parameter request, `0xA1` parameter response, `0xD8` disconnect | `research/molecular/transport_state_machines/tp20_state_machine.json` | ACCEPTED (opcode high nibble values only; the low-nibble sequence-number semantics are BLOCKED, EV-TP-04) |
| EV-TP-02 | Timings: channel setup 2 000 ms, ACK 500 ms, keepalive 1 000 ms | `tp20_state_machine.json` | ACCEPTED |
| EV-TP-03 | Channel setup addressing | `research/protocols/kwp2000.md` §2 says "setup CAN ID = 0x200 + logical address" with gateway `0x1F → 0x21F`; `tp20_state_machine.json` shows setup request bytes `01 C0 00 10 00 03 01` (destination address inside the payload); `research/protocols/vag_addressing.md` maps VAG `0x19` (gateway) → `0x21F` and `0x53` (parking brake) → `0x219` | CONFLICT — TP2.0 channel setup is non-executable until one addressing model is committed with source lines. |
| EV-TP-04 | Channel-parameter exchange (`A0`/`A1` payload: block size, T1–T4 encoding), packet sequence numbering, setup-response parsing (assigned TX/RX IDs) | not committed | BLOCKED_EVIDENCE — needed: TP2.0 communicator functions from `libCarista.so` with lines, **or** a captured PQ35 CAN trace (see `docs/testing/FIXTURE_SPEC.md` §4.1). |
| EV-KWP-01 | KWP services over TP2.0: `1A 9B`→`5A 9B` ECU info; `1A 9A`→`5A 9A` read coding; `3B 9A <data>`→`7B 9A` write coding / submodule coding; `18 02 FF 00` and `18 00 FF 00` → `58` DTCs; `12 <dtc> 04` → `52` freeze frame; `31 B8 <id>` / `32 B8 <id>` routines; `31 B9 <ch>`→`71 B9` select adaptation channel; `31 BA <ch>`→`71 BA` read adaptation; `31 BB <data>`→`71 BB` write adaptation | `research/protocols/kwp2000.md` §3 (lines 100905–105280) and `diagnostic_evidence.json` | ACCEPTED as request prefixes. Payload layouts after the prefix are BLOCKED_EVIDENCE per service (EV-KWP-03). |
| EV-KWP-02 | KWP `0x21` read / `0x27` write adaptation; `0x30` routine; `0x3B` "adaptation channel write" | `kwp2000_state_machine.json`; generated ShortAdaptation builder | REJECTED (Round 4 §4) |
| EV-KWP-03 | Response layouts: `5A 9B` identity fields, `5A 9A` coding bytes, `58` DTC records (2-byte DTC + status), `71 BA` adaptation value | `dtc.md` §3 gives `58 [Count] [DTC 2-byte + Status]`; others not committed | `58` layout ACCEPTED; all others BLOCKED_EVIDENCE — needed: `processPayloads`/parser functions with lines for `GetVagCanEcuInfoCommand`, `ReadVagCanLongCodingCommand`, `ReadVagCanAdaptationDataCommand`. |
| EV-KWP-04 | TP2.0 basic setting `31 <sub> <id>` with sub `0x01/0xB9`, `0x10/0xBA`, `0x19/0xBB` | `research/protocols/basic_settings.md` §2 | CONFLICT with EV-KWP-01 (`31 B9/BA/BB` are adaptation). |
| EV-KWP-05 | KWP measuring blocks `21 <group>` for live data | `research/protocols/live_data.md` §1 (no line) | BLOCKED_EVIDENCE — needed: request builder and parser functions with lines. |

## 8. Services and routines

| ID | Fact | Source | Status |
|---|---|---|---|
| EV-SV-01 | EPB (VAG `0x53`, UDS `0x752`/`0x7BC`): open `31 01 03 A1` / stop `31 02 03 A1`; close `31 01 03 A0` / stop `31 02 03 A0` | `diagnostic_evidence.json` `vag_uds_epb_*`; `research/protocols/service_routines.md` §1 | ACCEPTED |
| EV-SV-02 | EPB calibrate `31 01 03 A2` | `research/molecular/SERVICE_TOOL_INDEX.json` only | BLOCKED_EVIDENCE — needed: command class + line. |
| EV-SV-03 | DPF regeneration (engine `0x7E0`/`0x7E8`): `31 01 05 3D 04 00 00` stationary, `31 01 03 05 04 00 00` driving | `diagnostic_evidence.json` `vag_uds_dpf_regen_*` | ACCEPTED |
| EV-SV-04 | Routine status `22 01 02` → `62 01 02 <status>`; status high nibble: `0x0` none, `0x1` succeeded, `0x4` aborted (safety), `0x6` conditions not correct, `0x8` timeout, `0xC` in progress | `diagnostic_evidence.json` `vag_uds_routine_status_query`; `service_routines.md` §3 (`ReadVagUdsStatusCommand`, lines 110955–111035) | ACCEPTED. The alternative DID `0x0100` mentioned in `service_routines.md` is BLOCKED_EVIDENCE. |
| EV-SV-05 | Routine poll period 500 ms | `research/state_machines/service_routine_state_machine.md` (sequence diagram, no line) | ACCEPTED as policy value |
| EV-SV-06 | Routine overall timeout (30 000 ms for EPB) | `SERVICE_TOOL_INDEX.json` only | BLOCKED_EVIDENCE — `ServiceRoutineStateMachine` uses the plan's `maxDurationMs`; plans without an evidenced value are non-executable. |
| EV-SV-07 | Service preconditions (ignition on / engine off / voltage thresholds 12.0 V or 12.2 V / level ground) | `SERVICE_TOOL_INDEX.json`, service flow `.md` files (no lines) | BLOCKED_EVIDENCE as automotive facts. AutoDiag applies its own product safety policy (`WRITE_TRANSACTION_SPEC.md` §4) and shows the user checklist from pack metadata marked "advisory". |
| EV-SV-08 | Battery registration, service reset, TPMS flows | `research/molecular/service_flows/*.md` | BLOCKED_EVIDENCE — `CHATGPT_AUDIT_STATUS.json`: `CLASS_CONFIRMED_MAPPING_INCOMPLETE`. Needed: exact requests, DIDs, value encodings, security level and lines. |
| EV-SV-09 | UDS routine control `31 01|02|03 <id>` shape | ISO 14229-1 | OFFICIAL_PROTOCOL (shape only) |

## 9. Coding, adaptation, customizations

| ID | Fact | Source | Status |
|---|---|---|---|
| EV-CD-01 | UDS long-coding DID | `research/protocols/coding.md` §3 says `22 06 00` / `2E 06 00` (or `22 F1 A0`); all 438 `EXACT_VERIFIED` `VagUdsCodingSetting` variants in `research/molecular/SETTING_VARIANTS_INDEX.json` use `22 F1 A3` / `2E F1 A3`; Round 2 audit calls `F1A3` a dummy default | CONFLICT — every UDS coding variant with DID `0xF1A3` is `CONFLICT`. |
| EV-CD-02 | Protocol agreement for 240 exact variants (`command.protocol != ecu.protocol`) | `research/MOLECULAR_AUDIT_ROUND4.md` §3 | CONFLICT for each affected variant |
| EV-CD-03 | Multi-variant collapse `best = exact_v[0]` in `SETTING_INSTANCE_INDEX.json`, `SETTING_TO_COMMAND_MAP.json`, `READY_FEATURES.json` | Round 4 §2 | REJECTED — AutoDiag imports only `SETTING_VARIANTS_INDEX.json` (all variants) and never these three files. |
| EV-CD-04 | Extraction reproducibility (1826 of 1829 variants have no committed factory mapping) | Round 4 §1 | BLOCKED_EVIDENCE — every imported variant carries `reproducible = false` until `ghidracarista` commits a reproducibility attestation (`research/molecular/REPRODUCIBILITY_ATTESTATION.json`, format in `RUNTIME_PACK_SPEC.md` §9.3). Non-reproducible variants cannot be `EXACT`. |
| EV-CD-05 | Values recovered by the 50-instruction heuristic slice | Round 4 §5 | BLOCKED_EVIDENCE — `EXACT` requires per-variant dataflow provenance (`argument_provenance` with `method = "CFG_DATAFLOW"`). Current data has no such marker. |
| EV-CD-06 | Whitelist membership (258 distinct `VagWhitelists::*` symbols referenced by variants) | only symbol names are committed | BLOCKED_EVIDENCE — needed per whitelist: the list of part-number strings / SW-version ranges / ASAM IDs (`StringWhitelist`, `RangeWhitelist` contents) with lines. Without it, applicability of every whitelisted variant is incomplete. |
| EV-CD-07 | `research/db/ecu_variants.json` part-number patterns (e.g. `03G906*`) | no per-entry source | REJECTED as applicability evidence (no provenance). |
| EV-CD-08 | UDS adaptation variants: `22 <DID>` / `2E <DID>`, byte offset, bit mask, interpretation | `SETTING_VARIANTS_INDEX.json` (`VagUdsAdaptationSetting`, 304 exact) | ACCEPTED as **candidate** field values; executable only when EV-CD-04, EV-CD-05 and EV-CD-06 are resolved for the variant. |
| EV-CD-09 | Interpretations `ENABLED_DISABLED`, `YES_NO`, `YES_NO_INV`, `ON_OFF`, `ON_OFF_INV` | `SETTING_VARIANTS_INDEX.json` | ACCEPTED as codec mapping: set bit = enabled/yes/on, cleared bit = disabled/no/off; `_INV` inverts. |
| EV-CD-10 | TP2.0 coding `1A 9A` / `3B 9A` | EV-KWP-01 | ACCEPTED prefixes; coding payload layout BLOCKED (EV-KWP-03). |
| EV-CD-11 | TP2.0 adaptation flow select `31 B9 <ch>` → read `31 BA <ch>` → write `31 BB <data>` → read back | EV-KWP-01; Round 4 §4 | ACCEPTED as the only allowed flow shape; the value layout of `71 BA` and the `<data>` layout of `31 BB` are BLOCKED_EVIDENCE (EV-KWP-03). |

## 10. Live data

| ID | Fact | Source | Status |
|---|---|---|---|
| EV-LD-01 | OBD-II mode 01 PIDs `0x0C` (RPM, `(256A+B)/4`), `0x0D` (speed, `A` km/h) via `01 0C` / `01 0D` | `research/molecular/LIVE_DATA_INDEX.json` + SAE J1979 | OFFICIAL_PROTOCOL (read-only) |
| EV-LD-02 | Other 15 parameters in `LIVE_DATA_INDEX.json` | per-row `evidence` string | Imported per row: OFFICIAL_PROTOCOL when the row is an SAE J1979 PID with the J1979 formula; otherwise BLOCKED_EVIDENCE unless the row names a function and line. The compiler applies this rule mechanically (`RUNTIME_PACK_SPEC.md` §9.5). |
| EV-LD-03 | Multi-DID read `22 <DID1> <DID2> …` | `research/protocols/uds.md` §2.3 (`ReadVagUdsMultipleDataIdCommand`) | ACCEPTED as request shape; used only when a pack parameter group declares it. |

## 11. Consequence summary (derived, not a count to preserve)

- Read-only UDS diagnostics (discovery via `22 04 A1`, identity DIDs, DTC read/clear, OBD PIDs) are implementable now.
- EPB and DPF service routines on UDS are implementable now, gated by product safety policy.
- Everything that needs TP2.0 (PQ35 KWP diagnostics, coding, adaptation) is blocked by EV-TP-03 and EV-TP-04. The Golf 5 BLS acceptance target (master spec §61) therefore requires new evidence, most directly a captured CAN trace.
- No customization write can be `EXACT` today: EV-CD-04, EV-CD-05 and EV-CD-06 block every variant, and EV-CD-01 / EV-CD-02 additionally block most UDS coding variants. The runtime pack still carries all 1 829 variants, each classified.
