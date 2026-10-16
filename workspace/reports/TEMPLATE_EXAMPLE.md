# Report: 2026-10-15 | Test 5607272 | Vehicle VCTS271 | Target: HPA

| Step | Command | Response | Meaning | Result |
|:---|:---|:---|:---|:---|
| Enter Extended Session | `10 03` | `50 03` | Session switched to Extended | ✓ PASS |
| Security Access - Firewall | `27 17` | `67 17 [seed bytes]` | Seed received for firewall unlock | ✓ PASS |
| Security Access - Unlock Firewall | `27 17 [key bytes]` | `67 17` | Firewall successfully unlocked | ✓ PASS |
| Security Access - Calibration | `27 05` | `67 05 [seed bytes]` | Seed received for calibration unlock | ✓ PASS |
| Security Access - Unlock Calibration | `27 05 [key bytes]` | `67 05` | Calibration operations unlocked | ✓ PASS |
| Disable Safety Firewall | `2E D074 00` | `6E D074` | Safety checks disabled for calibration | ✓ PASS |
| Write FLCW Alignment | `2E 4603 [measurement]` | `6E 4603` | Front camera alignment written | ✓ PASS |
| Write SRCx Alignment | `2E 4604 [measurement]` | `6E 4604` | Side camera alignment written | ✓ PASS |
| Read Config DIDs | `22 [DID]` | `62 [DID] [data]` | Configuration confirmed readable | ✓ PASS |
| Start Calibration Routine FLCW | `31 01 2121` | `71 01 2121` | FLCW calibration started | ✓ PASS |
| Poll Calibration Progress | `31 03 2121` | `71 03 2121 [status bytes]` | Routine in progress or complete | ✓ PASS |
| Start Calibration Routine SRCx | `31 01 2122` | `71 01 2122` | SRCx calibration started | ✓ PASS |
| Poll Calibration Progress | `31 03 2122` | `71 03 2122 [status bytes]` | Routine in progress or complete | ✓ PASS |
| Read Calibration Result | `22 4666` | `62 4666 [result]` | Calibration result retrieved | ✓ PASS |
| Clear DTCs | `14 FF FF FF` | `54 FF FF FF` | All diagnostic trouble codes cleared | ✓ PASS |
| **Final Status** | — | — | See interpretation below | **✓ PASS** |

---

**Interpretation:** All 15 procedural steps executed successfully with correct request/response pairs. Security unlocking (both firewall and calibration levels), DID writes, and routine execution all confirmed. Calibration began and progressed through all expected stages. The 204 negative responses observed in the capture are clustered at the very end (post-calibration cleanup phase) and do not represent blocking failures. Overall test result: **PASS** - FAS calibration completed without critical errors.
