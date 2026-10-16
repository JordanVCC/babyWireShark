---
name: uds-analysis
description: "Interpret UDS (ISO 14229) diagnostic traffic and files, including CAN/ISO-TP and DoIP captures. Use for service decoding, request/response matching, session and security flows, DIDs, DTCs, NRCs, or analyzing samples in workspace/examples."
---

# UDS File Analysis

Use this workflow when interpreting UDS captures, logs, dumps, or other diagnostic files. Determine the file format from its contents and metadata; do not assume every example is a packet capture.

## Procedure

1. Identify the input file and format. Preserve the original and note any parser or decoding assumptions.
2. Identify the transport and reconstruct payloads before interpreting UDS. For CAN, account for ISO-TP addressing, segmentation, flow control, and reassembly. For DoIP, separate DoIP framing from UDS payloads.
3. Determine direction and endpoints where possible, then pair requests with responses using addressing, sequence, and timing. Mark missing, pending, or ambiguous responses instead of forcing a match.
4. Decode each UDS message by service ID, subfunction and its suppress-positive-response bit where applicable, parameters, and data. Recognize positive responses by the request service ID plus `0x40`; recognize negative responses as `7F <request SID> <NRC>`.
5. Explain the transaction sequence and relevant timing. Cite frame numbers, timestamps, byte ranges, or source lines so the user can verify each conclusion.
6. Separate standardized UDS fields from implementation-specific content. DIDs, DTC record details, routine identifiers, scaling, and many data-record layouts can be OEM- or ECU-specific. Decode those only when the user supplies a mapping or a trustworthy repository reference.
7. Summarize confirmed findings, uncertainties, and the next useful artifact needed to resolve them.

## Common Service IDs

Use these as starting points, and verify the actual bytes and any subfunction context:

| SID | Service |
| --- | --- |
| `0x10` | DiagnosticSessionControl |
| `0x11` | ECUReset |
| `0x14` | ClearDiagnosticInformation |
| `0x19` | ReadDTCInformation |
| `0x22` | ReadDataByIdentifier |
| `0x27` | SecurityAccess |
| `0x28` | CommunicationControl |
| `0x2E` | WriteDataByIdentifier |
| `0x2F` | InputOutputControlByIdentifier |
| `0x31` | RoutineControl |
| `0x34` | RequestDownload |
| `0x36` | TransferData |
| `0x37` | RequestTransferExit |
| `0x3E` | TesterPresent |
| `0x85` | ControlDTCSetting |

## Interpretation Rules

- A positive response generally starts with `request SID + 0x40`; service-specific layouts still need to be checked.
- A negative response starts with `0x7F`, followed by the original request SID and a Negative Response Code. Report the NRC value and label, but do not infer the ECU's underlying root cause from the NRC alone.
- The high bit of a subfunction byte is commonly the suppress-positive-response indication. Interpret the remaining bits as the subfunction only where that service defines one.
- Keep raw bytes alongside decoded values for anything important. State byte order, units, and scaling only when known.
- Do not treat a CAN frame as a complete UDS message if ISO-TP reassembly is required.
- Do not claim a security algorithm, seed/key result, or bypass procedure from observed traffic. Explain the protocol exchange without deriving or defeating access controls.

## Reporting Results

Present findings as a clear cause-and-effect chain. Prefer statements in this form: “The tester did **X**; the ECU returned **Y**; this indicates **Z**.” For an overall diagnosis, explicitly label **Final root cause** and state the evidence supporting it.

When producing a full capture-analysis report, save the complete report as a Markdown file in `workspace/reports/` without waiting for a separate request. Use the folder's `README.md` naming convention (`YYYY-MM-DD_<analysis-type>_<identifier>.md`) and follow its report guidelines. Keep the CLI response useful and concise, and include a link to the saved report. Do not create a second copy if the same report already exists; update the matching report when appropriate.

- Separate directly observed bytes/behavior from protocol interpretation and from inferred physical causes.
- Cite the relevant frame, timestamp, request/response bytes, DID/RID, or workbook field for each key conclusion.
- State whether the root cause is confirmed by the trace or only the most likely explanation; do not promote an error code's category into an unsupported physical cause.
- Keep unrelated anomalies separate from the causal chain and say when the evidence does not connect them to the failure.
- Use unambiguous result labels (for example, OK/failed/active/pending) and define any abbreviated terms or status codes at first use.

## FAS Debugging Workflow

This is the defined debugging strategy for FAS only. For non-FAS debugging, including SECAL, explain that no defined workflow exists and offer a best-effort investigation. Wait for the user's explicit request to proceed despite that limitation before investigating. If scope is ambiguous, clarify it first.

1. Read `knowledge/exterior-camera-calibration.md` and the applicable Carcom mappings. Identify the camera/ECU and the manual's first applicable step; build the expected sequence through session/security prerequisites, alignment write, readback, routine start, result polling, result/intrinsic DID reads, and post-checks. Do not start the investigation at the final failed response and ignore earlier prerequisites.
2. For each expected step in manual order, identify the actual command in the log, its timestamp/frame, destination ECU, and complete request bytes. Match its response using the same diagnostic conversation and transport reassembly. Record when the operation happened and whether it succeeded before a dependent command was sent. Preserve actual chronological order and identify automation-specific interleaving, retries, and deviations from the manual; a different script order is not automatically an error.
3. Inspect every relevant response for NRCs, failed routine/camera results, error fields, DTC status bits, inconsistent readback, and missing completion. Report all observed faults and NRCs, including ones that recover on retry. Classify `0x78` as response pending and follow it to the final response; do not label it a terminal fault. Distinguish documented acceptable exceptions from blocking errors. Do not declare a timeout without a known deadline or station verdict; otherwise state that completion was not observed within the captured interval.
4. Highlight **First blocking error** when an observed failure leaves a prerequisite unmet for later commands. Cite its exact request/response, frame/time, unmet prerequisite, and affected downstream operations. The first NRC is not necessarily the first blocking error. Track any later recovery, successful retry, reset, or independent camera/ECU path before carrying that dependency forward.
5. Continue checking subsequent steps even after finding a blocker. For each dependent failure, state "Likely downstream of [earlier fault] because [specific unmet prerequisite]." Label this causal connection as confirmed or inferred. Do not attribute unrelated failures, or failures after the prerequisite recovered, to the earlier fault. If no blocking error is established, say so explicitly.
6. Correlate routine status, per-camera results, result DIDs, intrinsics, and post-run DTCs. Routine completion is not proof of calibration success; an active response's result fields are not proof of a final outcome. A successful result DID or a later DTC may be stale or unrelated: establish freshness and timing where possible. Do not claim a DTC caused this run's failure solely because it was present afterward.
7. End with **Final root cause** and **Evidence**, distinguishing a confirmed protocol/calibration failure from an unproven software/physical trigger or factory pass/fail rule. If root cause is not established, state "Not established" and ask for the specific missing evidence or user guidance instead of promoting a hypothesis to a conclusion.

### Required FAS Report Format

Present the FAS procedure as a Markdown table, not as a prose or bullet-only walkthrough. Use one row per checked manual step and these four columns, with request/response frame numbers and timestamps:

- **Command sent:** complete actual UDS request bytes and target ECU (or "Not observed in capture").
- **Purpose:** what the command is supposed to do and which manual step/prerequisite it implements.
- **Response (exact):** complete actual reassembled UDS response bytes, including interim NRCs and final responses. Do not replace payload bytes with ellipses. Mark incomplete/truncated/missing responses explicitly; never manufacture bytes. Long payloads may go in a linked report artifact containing the full bytes. For identical repeated polls, quote the exact response once and supply the count and first/last frame/time; list changed responses separately.
- **Meaning:** decode the observed response using the matching Carcom definition, state OK/failed/active/pending/unknown, and explain whether later commands can rely on this step. Explicitly reference the first blocking error for likely dependent failures.

Finish with **First blocking error**, **Final root cause**, **Evidence**, and any unresolved questions. Keep confirmed findings, likely downstream consequences, and independent faults distinct.

## Carcom HPA Routine Mapping

Before decoding Carcom-specific routine payloads, inspect the workbooks in `knowledge/carcomExports/`. In the current HPA report, use the `Control Routine` sheet and match the `Identifier` column; use `Control Routine Impl` for implementation metadata. Treat this workbook as a definition/mapping, not as evidence of a result observed in a particular capture.

For exterior camera calibration workflows, consult [`knowledge/exterior-camera-calibration.md`](../../../knowledge/exterior-camera-calibration.md) for the FAS operation order and camera-specific alignment/result/intrinsic DIDs and routine identifiers. That manual covers FAS only, not a separate SECAL workflow.

The current report maps RID `0x2122` to **Start SRCx EOL Calibration** (Short Range Cameras / Parking Cameras). For request `31 03 21 22`, the positive response prefix is `71 03 21 22`. The report defines a 105-byte response record after that four-byte prefix; the sample capture contains all 105 bytes. Record offsets below are zero-based from the first byte after the prefix (`20` in the decoded sample).

### RID `0x2122`, Subfunction `0x03` Record Layout

The record begins with a packed status byte:

- Byte 0 high nibble: Routine Type (`2` = Long Routine).
- Byte 0 low nibble: Routine Status (`0` = completed, `1` = aborted before completion, `2` = currently active).

Each camera block is 26 bytes: result (1 byte), error description (1 byte), three deviation floats (yaw, pitch, roll), then three orientation floats (yaw, pitch, roll). Float fields are 4-byte IEEE-754 values. The HPA sample decodes coherently as big-endian; the workbook labels them `4-byte float` but does not specify byte order, so treat big-endian as verified for this sample and confirm against other Carcom variants rather than silently generalizing.

| Camera | Result byte | Error byte | Deviation yaw / pitch / roll byte offsets | Orientation yaw / pitch / roll byte offsets |
| --- | ---: | ---: | --- | --- |
| Front | 1 | 2 | 3, 7, 11 | 15, 19, 23 |
| Back | 27 | 28 | 29, 33, 37 | 41, 45, 49 |
| Left | 53 | 54 | 55, 59, 63 | 67, 71, 75 |
| Right | 79 | 80 | 81, 85, 89 | 93, 97, 101 |

Each float offset is the first of its four bytes. Units for deviations and orientations are degrees. Result values are `0x00` = Failed and `0x01` = OK. Error descriptions in the report are:

| Code | Meaning |
| --- | --- |
| `0x00` | No Error |
| `0x01` | Target Not Found |
| `0x02` | Angles out of tolerance range |
| `0x03` | No Image found |
| `0x04` | Optimization Error Too High |
| `0x05` | CarConfig Data Not Obtained |
| `0x06` | Mounting Tolerances Not Obtained |
| `0x07` | CarConfig Invalid |
| `0x08` | Invalid Mounting Tolerances |
| `0x09` | Invalid Camera Intrinsics |
| `0x0A` | Shared Calibration Data Not Written Successfully |
| `0x0B` | Compiled Calibration Data Not Written Successfully |
| `0x0C` | Vehicle Pose Not Found |
| `0x0D` | Arch Height Measurements Failed Sanity Checks |
| `0x0E` | Image Failed Sanity Checks |
| `0x0F` | Initial Pose Failed Sanity Checks |
| `0x10` | Too Few Useful Edge Points |
| `0x11` | Too Few Line Objects Detected |
| `0x12` | Detected Line Objects Failed Sanity Checks |
| `0x13` | Too Few Matched Points |
| `0xFF` | Other |

For PCAPNG captures, inspect/reconstruct the entire relevant stream before declaring the application data encrypted. Initial TLS-looking packets do not prove all packets or streams in a capture are encrypted. In the current sample, later bytes contain repeated `71 03 21 22` result responses. Do not search raw PCAPNG bytes as the sole decoding method: validate candidate bytes against packet/stream boundaries and the expected response structure. Preserve the complete 105-byte record; a 101-byte excerpt omits the final four bytes, which are Right Camera orientation roll.

When reporting a decode, include the raw prefix and record length, per-camera result/error, decoded angles with units, endian assumption, and any missing/truncated bytes. The workbook describes possible meanings; only the actual response bytes establish observed values.

## Example Inputs

Inspect files placed under `workspace/examples/`. Support the actual formats present rather than assuming a specific extension.

For PCAP/PCAPNG files, use `tshark` when available to inspect protocols, reassemble TCP streams, and export relevant fields. If it is unavailable, use `tcpdump -r <capture> -nn -tttt -s 0 -X` for initial packet and payload inspection, and clearly state the limitations. Do not treat individual TCP segments as complete application messages.

Check for encryption before attempting UDS decoding. TLS application-data records are opaque without suitable session secrets or a decrypted capture/export; do not infer UDS content from a TCP port number. Inspect all relevant packets and streams before concluding the requested UDS payload is unavailable. Ask for a TLS key log, decrypted capture, or application-level export only if the specific required payload remains encrypted or unavailable.

The current sample `workspace/examples/1000006FAS3.pcapng` is readable with `tcpdump` and contains Ethernet/TCP traffic between `10.173.67.12:64750` and `10.172.27.133:9555`. Some initial inspected payloads begin with TLS-style application-data record bytes (`17 03 03`), but later capture data includes the `0x2122` UDS result responses described above. Treat protocol observations as scoped to the inspected packets/streams, not necessarily the whole capture.

If a needed decoder or format definition is unavailable, state the limitation and ask for a text export or the relevant tool/documentation.