# Report: 2026-10-16

**Capture:** `workspace/Logs/1000008_FAS2_FASPrerel_Run2.pcapng` (test 1000008 / FAS2 / FASPrerel Run2)
**Type:** FAS exterior-camera calibration debug (HPA, FLCW + SRCx)

## Test Metadata

| Item | Value |
|:---|:---|
| Capture time | 2026-10-13 15:53:14 – 16:01:20 (486.3 s). All times below are seconds relative to capture start. |
| Transport | DoIP (TCP/13400) → UDS, tester `10.173.67.13` → gateway `10.173.112.60`. Frames reassembled per TCP stream/direction and split on DoIP headers (payload type 0x8001); `tshark` UDS dissector crashed on a few frames, so decoding was done from reassembled bytes. |
| Calibration stream | TCP stream 122 (port 53027), t=239.8–425.8 s. Stream 9 (t=18–60 s) is an earlier connection (session/security/DTC-clear/DTC-read, `2E D1 34 02` rejected). |
| Tester logical address | `0x0E80` |
| Target ECUs (as addressed) | `0x1D12` = HPA (alignment DIDs/RIDs `0x2121`/`0x2122` answer here); `0x1D31`–`0x1D34` = camera ECUs (RID `0x2046`, `0x2135`); `0x1C01`, `0x1D89`, `0x1D90`, `0x1451` = other ECUs. Address→role mapping is inferred from behavior, not from a supplied table. |
| Also in file | SMB, HTTPS, other TCP/9555 flows – unrelated to UDS. |

Calibration mapping used: `knowledge/exterior-camera-calibration.md` (FLCW: DID `0x4603`, RID `0x2121`, result DID `0x4666`; SRCx: DID `0x4604`, RID `0x2122`, result DID `0x467E`, intrinsics `0x467F`). RID `0x2122` layout from the Carcom HPA report (big-endian floats, degrees).

## Procedure Stages (stream 122, target HPA `0x1D12`)

Alignment values written: FL `031D`=797 mm, FR `0326`=806 mm, RL `02C7`=711 mm, RR `0352`=850 mm (all in the manual's 500–1200 mm range).

| # | Command sent (frame / t) | Purpose | Response (exact) | Meaning |
|:--|:---|:---|:---|:---|
| 1 | `10 03` → 1D12 (2554 / 239.902, again 2560) | Extended session | `50 03 00 32 01 F4` (2555 / 239.918, 2561) | OK. |
| 2 | `27 17 00 01 00 01 2E 90 56 77 …` → 1D12 (2566 / 240.452); then `27 18 00 03 95 78 A7 FD …` (2571 / 240.474) | SecurityAccess level 0x17 (request/response carry long, signature-style payloads – not interpreted) | `67 17 00 02 FF 28 61 B1 …` (2569), then `67 18` (2578 / 240.508) | OK – level 0x17 unlocked. Full long payloads in raw capture. |
| 3 | `2E D0 74 00` (2580 / 240.615) | Disable firewall | `6E D0 74` (2581 / 240.660) | OK (no NRC 0x31 firewall-absent case). |
| 4 | `10 03`, `27 03 …`/`27 04 …`, `2E D1 34 00` (2593–2620 / 241.2–241.29) | Session/security + write DID `D134` (meaning not in repo knowledge) | `50 03 00 32 01 F4`, `67 03 …`, `67 04`, `6E D1 34` (2623 / 241.316) | OK. In the earlier stream 9, `2E D1 34 02` (frame 801 / 44.690) got `7F 2E 22` (conditionsNotCorrect) – different connection, ~200 s before the run; not connected to calibration by this evidence. |
| 5 | `10 03` (3154 / 262.304), `27 05 00 01 00 01 B4 58 …` (3165), `27 06 00 03 B2 E8 …` (3180) | Level 0x05 unlock | `50 03 00 32 01 F4` (3163), `67 05 00 02 86 5A …` (3173), `67 06` (3187 / 262.377) | OK – level 0x05 unlocked. |
| 6 | `2E 46 03 0B D9 05 78 03 1D 03 26 02 C7 03 52` (3189 / 262.377) | Write FLCW alignment | `6E 46 03` (3198 / 262.397) | OK. Note: fixed prefix is `0B D9 05 78`; the manual lists `0B 0A 05 78`. Station may use a different configuration – deviation noted, no NRC. |
| 7 | `22 46 03` (3200 / 262.40) | Read back FLCW alignment | `62 46 03 0B D9 05 78 03 1D 03 26 02 C7 03 52` (3219 / 262.443) | OK, matches write. |
| 8 | `2E 46 04 03 1D 03 26 02 C7 03 52` (3221 / 262.444) | Write SRCx alignment | `6E 46 04` (3227 / 262.455) | OK. |
| 9 | `22 46 04` (3229 / 262.455) | Read back SRCx alignment | `7F 22 78` (3254 / 262.511) then `62 46 04 03 1D 03 26 02 C7 03 52` (3256 / 262.514) | `0x78` response pending, then OK, matches write. |
| 10 | `10 03`, `27 05 …`, `27 06 …` → 1D12 (3454–3464 / 266.60–266.64) | Re-enter session/unlock | `50 03 …`, `67 05 00 00 …`, (level-0x05 flow) | OK. |
| 11 | `31 01 21 21` (3466 / 266.638) | Start FLCW routine | `71 01 21 21 22` (3471 / 266.661) | Started. |
| 12 | `31 03 21 21` (3473 / 266.66, 3489 / 266.93) | Poll FLCW | `71 03 21 21 22 01 00 00 00 …(00) C2 B4 00 00 00 00 00 00 …` (3478 / 266.674, active, status 0x2); `71 03 21 21 20 01 00 BF 37 B9 59 3F CB 66 AF 3F BB 2E DC BF 37 B9 59 3F AD F6 0C 3F BB 2E DC 00 00 00 00` (3490 / 266.935, completed) | Routine completed (status 0x0), 31-byte record; result byte `01`, error `00`. FLCW record layout is not defined in the available mapping (only `0x2122` is), so only the first bytes are interpreted. |
| 13 | `22 46 66` (3492 / 266.94) | FLCW result DID | `62 46 66 20 01 00 BF 37 B9 59 3F CB 66 AF 3F BB 2E DC BF 37 B9 59 3F AD F6 0C 3F BB 2E DC 00 00 00 00` (3493) | Identical to routine result. FLCW OK. |
| 14 | **Attempt 1** `10 03`, `27 05/06`, `31 01 21 22` (3509 / 267.577) | Start SRCx routine | `71 01 21 22 22` (3512 / 267.602) | Started. |
| 15 | `31 03 21 22` polls (first 3514 / 267.61 … ) | Poll SRCx | Active: `71 03 21 22 22` + 105-byte record (all four cameras `01 00 …`, orientation roll/pitch placeholder C2B40000) (3515 / 267.611) – repeated, last at 3550 | Active – record not a result. |
| 16 | `31 03 21 22` (3547 / 268.62) | Poll | **Final** (3550 / 268.657): `71 03 21 22 20 00 10 B6 18 1E 0B C2 C6 05 1E 43 33 FF FF 00 00 00 00 C2 B4 00 00 00 00 00 00 01 00 BF 23 FA 00 BF 21 37 41 C0 0D 7C 06 43 33 5C 07 41 55 EC 8A C0 0D 7C 0A 01 00 3F 20 02 01 BF 85 15 0E 3E BA 62 D9 42 B5 40 03 42 1B D7 55 C3 33 A2 CD 00 10 42 B4 00 00 C3 02 00 00 43 33 FF FF 00 00 00 00 C2 B4 00 00 00 00 00 00` | Routine completed (status 0x0) but **Front: result 0 (Failed), error `0x10` Too Few Useful Edge Points; Back OK; Left OK; Right: Failed, error `0x10`.** |
| 17 | `22 46 7E` (3552 / 268.66) | SRCx result DID | `62 46 7E 20 00 10 B6 18 …` (3553) – same 105-byte record (frame 3553/3558, 108 B incl. DID) | Confirms failure (identical data). |
| 18 | **Attempt 2** `10 03`, `27 05/06`, `31 01 21 22` (3578 / 268.772) → `71 01 21 22 22` (3587); final (3613 / 269.847) | Retry | Final record has Front `00 10`, Back `01 00`, Left `01 00`, Right `00 10` (`22 46 7E` 3625: same; `7F 22 78` pending at 3623 / 269.905) | Same failure. |
| 19 | **Attempt 3** start (3641 / 269.977) | Retry | `7F 31 78` (3644 / 270.037), then `71 01 21 22 22` (3646); polls `7F 31 78` (3651), final (3762 / 271.229) | Same failure; Front/Right `00 10`. |
| 20 | **Attempt 4** start (3793 / 271.292) | Retry | `71 01 21 22 22` (3794); `7F 31 78` (3799 / 271.365); final (3815 / 272.483) | Same failure; Front/Right `00 10`. |
| 21 | `27 21`/`27 22` (unlock, 3833–3841), `22 47 69` (3843 / 272.628) | Not in manual FAS sequence (unknown DID mapping) | `62 47 69 3F ED 65 AF 3E 9C 4C E1 …` (3844) | Positive; meaning not decodable with the supplied mapping. |
| 22 | `22 46 7F` (SRCx intrinsics) | Manual post-check | **Not observed in capture** | Missing – not read after any attempt. |
| 23 | `14 D4 D5 54 …` clear DTCs (6272–6330 / 422.27–422.38); `19 02 20`, `19 03`, `19 06 …`, `19 04 …` (6427–6624 / 423.8–424.6) | Post-check | `54` positive responses; DTC list in `59 02 FF …` | See DTC section. |

## SRCx result decode (final response, frame 3550 – attempt 1; attempts 2–4 are equivalent)

Layout: RID `0x2122` Carcom record, 105 bytes after prefix `71 03 21 22`; byte 0 = `0x20` (Long routine, status 0 = completed). Big-endian floats, degrees.

| Camera | Result | Error | Deviation yaw/pitch/roll | Orientation yaw/pitch/roll |
|:---|:---|:---|:---|:---|
| Front | 0 = Failed | `0x10` Too Few Useful Edge Points | −0.0 / −99.01 / 180.0 | 0.0 / −90.0 / 0.0 |
| Back | 1 = OK | `0x00` | −0.641 / −0.630 / −2.211 | 179.359 / 13.370 / −2.211 |
| Left | 1 = OK | `0x00` | 0.625 / −1.040 / 0.364 | 90.625 / 38.960 / −179.636 |
| Right | 0 = Failed | `0x10` Too Few Useful Edge Points | 90.0 / −130.0 / 180.0 | 0.0 / −90.0 / 0.0 |

Per-attempt results (final frames): 3550 (268.66 s), 3613 (269.85 s), 3762 (271.23 s), 3815 (272.48 s) – identical result/error pattern each time; Back/Left deviations stable to ~0.02°. The Front/Right angle values are default-like placeholders (not a valid pose); they should not be read as measurements.

## Error Analysis

Observed negative responses in the calibration window (stream 122):

| NRC | Frame / t | ECU | Class | Effect |
|:---|:---|:---|:---|:---|
| `7F 22 78` | 3254 / 262.511; 3623 / 269.905 | 1D12 | Response pending → resolved by positive `62` (3256, 3625/3628) | None |
| `7F 31 78` | 3644 / 270.037, 3651 / 270.12, 3799 / 271.365 | 1D12 | Response pending → resolved by `71 01`/`71 03` | None |
| `7F 27 78` | 3232 / 262.469, 3265 / 262.664 | 1C01 | Pending → resolved | Unrelated ECU |
| `7F 27 78` | 3683, 3694 (1D89); 3700, 3710 (1D90) | other ECUs | Pending → resolved | Unrelated |
| `7F 2E 78` | 4080, 4937–4964 | 1D89, 1D31–1D34 | Pending → resolved (`6E`) | Unrelated |
| `7F 31 78` | 4087 / 282.435 | 1D90 | Pending → resolved | Unrelated |
| `7F 14 78`, `7F 19 78` | 42.3–43.7 s (stream 9) | multiple | Pending → resolved | Pre-test cleanup |
| `7F 2E 22` | 801→803 / 44.690–44.701 | 1D12 | conditionsNotCorrect (terminal) | Stream 9 `D134=02` write rejected; later stream 122 `D134=00` and `=02` writes succeeded (6717 / 425.327). Not tied to calibration. |

No terminal NRC occurred inside the calibration sequence. The failure is carried in the routine **result bytes**, not in a UDS negative response.

### DTCs (HPA `0x1D12`, `19 02 FF`)
Before (frame 567 / 43.5 s) vs after (frame ~6431 / 423.8 s): the set includes DTC status `0x2F` entries (`D4D554`, `D50954`, `D50A54`, `D50B54`, `D50C54` before; `D50A55`, `D50C55`, `F04A51` after) and `0x26/0x27/0x38` entries. DTC definitions are not in the supplied mapping, and the DTCs were cleared at 422 s, after the calibration attempts; the post-run `D50A55`/`D50C55` (status `0x2F`, test failed) cannot be attributed to the Front/Right failure from this capture.

## Conclusions

**First blocking error:** SRCx routine RID `0x2122` final poll, frame 3550 / t=268.657 s (`31 03 21 22` → `71 03 21 22 20 00 10 …`): routine completed but Front and Right cameras returned Failed with error `0x10` (Too Few Useful Edge Points). Unmet prerequisite: valid SRCx calibration for Front/Right; the manual expects routine success status 0 with a usable result, then result/intrinsics reads. Prerequisites before it (session `10 03`, SecurityAccess `0x17`/`0x05`, firewall disable, alignment writes and readbacks, FLCW routine) all completed OK.

**Likely downstream of that error (confirmed by identical results):** the three retries (frames 3578, 3641, 3793) reproduced the same Front/Right failures; the result DID `22 46 7E` mirrors the routine record (confirmed). Intrinsics read (`22 46 7F`) was not observed – tester did not proceed after the fourth failure (cause of not reading is inferred, not shown).

**Independent/unrelated items:** `0x78` pending responses (all resolved), `2E D1 34 02` → `7F 2E 22` in the earlier connection, other ECU routines (`0x2046`, `0x2135`, `0x3074`, `0xDC02` …), and the post-run DTC list.

**Final root cause (confirmed at protocol level):** In every SRCx run, the HPA reported routine `0x2122` as completed with Front and Right camera results Failed, error `0x10` Too Few Useful Edge Points; Back and Left passed. The FLCW routine `0x2121` succeeded.
**Not established:** why the edge points were insufficient for Front/Right (target visibility/placement, image quality, vehicle position, software) – the trace gives no evidence for a physical or software trigger, and the station pass/fail rule is not available.

**Evidence:** frames 3550, 3613, 3762, 3815 (final polls); 3553/3625/3765/3820 (`22 46 7E` readback); alignment write/readback frames 3189–3256 OK; FLCW result frames 3490/3493.

**Unresolved / needed:** camera images or DLT logs for the Front/Right cameras during the run; mapping for DID `0x4769`, `0xD134`, RID `0x2046`/`0x2135`; DTC definitions for `D50A55`/`D50C55`; FLCW `0x2121` record layout; confirmation that the `0B D9 05 78` FLCW prefix is correct for this configuration.

## Raw Data
Reassembled UDS messages were extracted with `tshark` fields (`tcp.payload`) and parsed from DoIP framing; full byte strings above are complete except the long SecurityAccess seed/key payloads, which are truncated with `…` (not interpreted; available in the capture).
