# Exterior Camera Calibration Reference

Distilled from the ART AD/ADAS Platform **Manual FAS Calibration Procedure**, document version 2.0. The source PDF was supplied temporarily under `knowledge/InputKnowledge/`; this reference is self-contained so that temporary folder can be removed.

## Scope

This manual documents factory calibration (FAS) sequences for FLCW, SRCx, LRBx, and LRIx. It does not define a separate service calibration (SECAL) procedure. Do not treat the FAS workflow below as SECAL; add SECAL steps only when a service-calibration source is provided.

## Shared FAS Procedure

Follow this order for a camera procedure:

1. Measure wheel-arch heights and convert each millimeter value to a four-digit hexadecimal value. The manual gives 828 mm = `033C`, 830 mm = `033E`, and 850 mm = `0352`; typical values are 500–1200 mm.
2. Perform the ECU-specific security/session steps listed below.
3. Write the alignment data to that camera's alignment DID. Preserve the specified fixed prefix bytes and append the four measurements in this exact order: **Front Left, Front Right, Rear Left, Rear Right**.
4. Read back/confirm the alignment DID.
5. Start the calibration RID with RoutineControl subfunction `0x01` (`31 01 <RID>`); poll results with subfunction `0x03` (`31 03 <RID>`). The manual says to look for routine success status `00`.
6. Read the calibration-result DID and intrinsic-parameters DID.
7. Verify the generated calibration file(s) and confirm DTC status. Save verified files/logs as required by the station procedure.

## Camera Commands

The commands below are UDS request bytes as shown by the manual. `<FL> <FR> <RL> <RR>` are four-digit wheel-arch measurements in the ordering above, serialized as bytes. For example, 830/828/830/830 mm is `03 3E 03 3C 03 3E 03 3E`.

| Camera procedure | Alignment write and confirm | Start and poll routine | Post-calibration reads | File noted by manual |
| --- | --- | --- | --- | --- |
| FLCW (front camera) | Write `2E 46 03 0B 0A 05 78 <FL><FR><RL><RR>`; confirm `22 46 03` | RID `0x2121`: start `31 01 21 21`; poll `31 03 21 21` | Result `22 46 66`; intrinsics `22 46 7D` | `/var/zenseact`, `/var/fs/zen_qm_vision` |
| SRCx (surround view) | Write `2E 46 04 <FL><FR><RL><RR>`; confirm `22 46 04` | RID `0x2122`: start `31 01 21 22`; poll `31 03 21 22` | Result `22 46 7E`; intrinsics `22 46 7F` | `/var/fs/zen_sense_a` |
| LRBx (rear-side cameras) | Write `2E 47 34 13 39 09 59 04 4C 41 9C 00 00 00 <FL><FR><RL><RR>`; confirm `22 47 34` | RID `0x2123`: start `31 01 21 23`; poll `31 03 21 23` | Result `22 47 35`; intrinsics `22 47 36` | `/var/fs/zen_sense_b` |
| LRIx | Write `2E 4B 8E 03 83 09 EB 03 52 00 00 00 00 <FL><FR><RL><RR>`; confirm `22 4B 8E` | RID `0x026E`: start `31 01 02 6E`; poll `31 03 02 6E` | Result `22 4A 92`; intrinsics `22 4B 8F` | Not specified in the procedure section |

For SRCx, RID `0x2122`'s detailed response field map is in this repository's Carcom HPA report and in the UDS analysis skill. The other RIDs must be decoded using their own report mappings; do not reuse the `0x2122` response layout for them.

## Security and Preconditions

- HPA: enter Extended Session `10 03`; the procedure shows SecurityAccess level `0x17` (`27 17 ...` / key response `27 18 ...`), firewall disable `2E D0 74 00`, then level `0x05` (`27 05 ...` / key response `27 06 ...`). The manual's quick-reference table distinguishes level `0x17` as automatic via DSA file transfer and level `0x05` as requiring manual unlock, but a separate note broadly says HPA security is automatic/no manual unlock. Preserve this source inconsistency and confirm the actual station/ECU behavior; do not assume both levels are automatic.
- If firewall disable returns NRC `0x31` because the vehicle has no firewall, the manual says this is acceptable and to proceed.
- HPB: manual SecurityAccess level `0x05` is required before calibration; the LRBx procedure notes this is usually on HPB and may require a separate connection/security setup.
- Use the matching SDBB for the calibration. For REL12, the manual notes to use 726A with HPA/HPB added.

## Verification and Diagnostic Notes

- The manual says to verify calibration files exist before proceeding and that DTCs should show passed status after successful calibration. Its DTC status table labels `0x2F` Test Failed (Active), `0x50` Test Passed (Inactive), `0x00` Test Not Completed, and `0x2E` Test Completed - Passed This Cycle.
- Listed common NRC guidance: `0x10` General Reject (system not ready); `0x22` Conditions Not Correct (camera info not received or image cannot be acquired); `0x31` Request Out Of Range (invalid/unregistered DID or RID). Treat these as manual guidance, not proof of the ECU's underlying cause.
- The manual recommends collecting DLT logs after calibration. For raw camera image dumps, it documents ImageJ import as 16-bit unsigned, little-endian: FLCW/LRBx/FLCI dimensions 3848 x 2168; SRCx dimensions 1936 x 1220.

## Procedure Gaps

No SECAL/service-calibration sequence is included in this FAS manual. It does not establish the correct SECAL DID/RID, security level, or order of operations. Keep SECAL analysis separate pending an applicable service-calibration document.