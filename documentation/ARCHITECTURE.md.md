# MIL-STD-188-203-1A & NATO STANAG 5511 (Link 11\) Technical Architecture

## 1\. Executive Summary & Specification Scope

Link 11 (TADIL-A), standardized by the US Department of Defense in MIL-STD-188-203-1A and by NATO in STANAG 5511, is a secure, half-duplex tactical digital information link operating over High Frequency (HF, 2–30 MHz) and Ultra High Frequency (UHF, 225–400 MHz) bands. It enables real-time tactical data exchange between naval surface combatants, airborne early warning aircraft, maritime patrol aircraft, and shore command centers.

### Key Architectural Concepts:

1. **Network Operation Modes**:  
   - **Roll-Call Mode**: Primary operating mode. The designated Net Control Station (NCS) sequentially polls participating picket units (PU) in a continuous round-robin cycle. Each picket unit responds by transmitting its accumulated tactical tracks.  
   - **Broadcast Mode**: Single unit continuously transmits without polling responses.  
   - **Short Broadcast & Net Sync Modes**: Used for radio channel synchronization and propagation calibration.  
2. **30-Bit Word Structure**:  
   - 24 bits of tactical data payload.  
   - 6 bits of modified Hamming parity code generated via generator matrices, providing Single Error Correction and Double Error Detection (SEC-DED).  
3. **M-Series Message Dictionary**:  
   - `M.1`: Own Unit Position & Status (picket address, course, speed, ownship coordinates).  
   - `M.2`: Air Track Report (track number, identity, 100-ft altitude steps, speed, heading).  
   - `M.3`: Surface Track Report (maritime surface contacts, ship identification).  
   - `M.4`: Subsurface Contact Report (sonar contacts, submarine depth).  
   - `M.9`: Command & Engagement (target vectors, missile assignment, intercept commands).  
4. **Physical Layer Modulation (CLEW & SLEW)**:  
   - **CLEW**: Conventional Link Eleven Waveform using 16 differential audio tones (1364 bps or 2250 bps).  
   - **SLEW**: Single-tone Link Eleven Waveform using single-carrier PSK (1800 bps) with increased HF fade resistance.