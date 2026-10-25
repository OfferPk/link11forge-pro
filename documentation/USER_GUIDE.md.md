# Link11Forge Pro User Guide & Developer Manual

## 1\. Quickstart Guide

1. Launch `index.html` in your web browser.  
2. In the "Naval Tactical Data Link Config" panel, enter your Picket Address in octal format (e.g. `012` octal).  
3. Select your Station Role (`Picket Station` or `Net Control Station NCS`).  
4. Choose the Link 11 Waveform Mode (`CLEW Fast 2250 bps`, `CLEW Slow 1364 bps`, or `SLEW 1800 bps`).  
5. Select target M-Series message type (e.g. `M.1 Own Unit Position` or `M.2 Air Track Report`).  
6. Set tactical track parameters (Track \#, Identity e.g. Hostile, Altitude e.g. `25,000 ft`).  
7. Click **Encode 30-Bit Word** to generate the 24-bit data word, compute the 6-bit modified Hamming parity code, and inspect the bitfield dissector.  
8. Click **Run NCS Poll Step** to simulate a single round-robin interrogation in the roll-call network cycle and monitor net cycle time.  
9. Click **Run Normative Audit** to evaluate all 10 MIL-STD-188-203-1A protocol compliance checks.

## 2\. Standalone Headless Python CLI

Run automated regression testing and compliance verification:

python3 link11-cli.py \--audit \--picket 012 \--msg M2 \--track 42