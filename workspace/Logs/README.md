# UDS Examples

Place sample captures, logs, or diagnostic files in this folder for the Repository Agent to inspect. It will first identify each file's format and transport, then analyze the UDS messages it can reliably decode.

Keep original files unchanged. If a file contains OEM-specific DIDs, routines, DTC details, or scaling, include the relevant mapping/documentation as a separate file when available; otherwise those payloads may remain undecoded.

The current `1000006FAS3.pcapng` is readable as Ethernet/TCP by `tcpdump`. The inspected packets contain TLS-style application-data records, so UDS contents cannot be decoded from the encrypted payload without appropriate TLS secrets or a decrypted/application-level export. A destination port alone is not evidence that the traffic is UDS.