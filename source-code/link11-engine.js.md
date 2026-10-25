/\*\*  
 \* MIL-STD-188-203-1A & NATO STANAG 5511 Tactical Data Link (Link 11 / TADIL-A) Core Engine  
 \* Implements 24-bit M-series message bitfield packing, 6-bit modified Hamming parity code,  
 \* Net Control Station (NCS) roll-call polling cycle, and CLEW/SLEW timing modeling.  
 \*/

// Link 11 Message Labels (M-Series Dictionary)  
const LINK11\_MESSAGE\_LABELS \= {  
  M1\_OWN\_UNIT: 0x01,     // M.1: Own Unit Position & Status  
  M2\_AIR\_TRACK: 0x02,    // M.2: Air Track Position & Identity  
  M3\_SURFACE\_TRACK: 0x03,// M.3: Surface Track Report  
  M4\_SUBSURFACE: 0x04,   // M.4: Subsurface Contact Report  
  M9\_COMMAND: 0x09       // M.9: Command & Weapon Assignment  
};

// Track Identity  
const TRACK\_IDENTITIES \= {  
  PENDING: 0x0,  
  UNKNOWN: 0x1,  
  ASSUMED\_FRIEND: 0x2,  
  FRIEND: 0x3,  
  NEUTRAL: 0x4,  
  SUSPECT: 0x5,  
  HOSTILE: 0x6  
};

// Link 11 Waveform Modes  
const WAVEFORM\_MODES \= {  
  CLEW\_SLOW: { name: 'CLEW Slow (1364 bps)', baud: 1364, bitsPerSecond: 1364 },  
  CLEW\_FAST: { name: 'CLEW Fast (2250 bps)', baud: 2250, bitsPerSecond: 2250 },  
  SLEW:      { name: 'SLEW Single-Tone (1800 bps)', baud: 1800, bitsPerSecond: 1800 }  
};

class Link11Engine {  
  constructor(picketAddress \= 0o12, isNcs \= false, waveform \= 'CLEW\_FAST') {  
    this.picketAddress \= picketAddress; // Octal Picket Address (e.g. 012 octal)  
    this.isNcs \= isNcs;                 // Net Control Station flag  
    this.waveform \= WAVEFORM\_MODES\[waveform\] \|\| WAVEFORM\_MODES.CLEW\_FAST;  
    this.rollCallPickets \= \[0o12, 0o14, 0o16, 0o20\]; // Pickets in net  
    this.currentPolledIndex \= 0;  
    this.cycleCount \= 0;  
  }

  /\*\*  
   \* Computes 6-bit modified Hamming code parity for a 24-bit data word  
   \* Provides Single Error Correction and Double Error Detection (SEC-DED).  
   \*/  
  static calculateHammingCode(data24) {  
    let p0 \= 0, p1 \= 0, p2 \= 0, p3 \= 0, p4 \= 0, p5 \= 0;

    for (let i \= 0; i \< 24; i++) {  
      const bit \= (data24 \>\> i) & 1;  
      if (bit) {  
        if ((i \+ 1\) & 1\)  p0 ^= 1;  
        if ((i \+ 1\) & 2\)  p1 ^= 1;  
        if ((i \+ 1\) & 4\)  p2 ^= 1;  
        if ((i \+ 1\) & 8\)  p3 ^= 1;  
        if ((i \+ 1\) & 16\) p4 ^= 1;  
        p5 ^= 1; // Overall parity  
      }  
    }

    const hamming6 \= (p5 \<\< 5\) \| (p4 \<\< 4\) \| (p3 \<\< 3\) \| (p2 \<\< 2\) \| (p1 \<\< 1\) \| p0;  
    return hamming6 & 0x3F;  
  }

  /\*\*  
   \* Encodes a 30-bit Link 11 Transmission Word (24 bits Data \+ 6 bits Hamming Parity)  
   \*/  
  static encodeWord30(data24) {  
    const parity6 \= Link11Engine.calculateHammingCode(data24);  
    // 30-bit word: Data(24b) shifted left by 6 \+ Parity(6b)  
    const word30 \= ((data24 & 0xFFFFFF) \<\< 6\) \| (parity6 & 0x3F);  
    return {  
      data24Hex: '0x' \+ (data24 & 0xFFFFFF).toString(16).toUpperCase().padStart(6, '0'),  
      parity6Hex: '0x' \+ parity6.toString(16).toUpperCase().padStart(2, '0'),  
      word30Hex: '0x' \+ (word30 \>\>\> 0).toString(16).toUpperCase().padStart(8, '0'),  
      word30: word30  
    };  
  }

  /\*\*  
   \* Encodes an M.1 Own Unit Position Report Word  
   \* \[23:20\] Label (0x1 \= M.1)  
   \* \[19:14\] Picket Address (6 bits)  
   \* \[13:7\]  Course (7 bits, 0-127 mapped to 0-360 deg)  
   \* \[6:0\]   Speed (7 bits, 0-127 knots)  
   \*/  
  encodeM1OwnUnit(courseDeg \= 180, speedKts \= 25\) {  
    const course7 \= Math.round((courseDeg / 360.0) \* 127\) & 0x7F;  
    const speed7 \= Math.min(127, Math.max(0, speedKts)) & 0x7F;

    let data24 \= 0;  
    data24 \|= (LINK11\_MESSAGE\_LABELS.M1\_OWN\_UNIT & 0x0F) \<\< 20;  
    data24 \|= (this.picketAddress & 0x3F) \<\< 14;  
    data24 \|= (course7 & 0x7F) \<\< 7;  
    data24 \|= (speed7 & 0x7F);

    const encoded \= Link11Engine.encodeWord30(data24);

    return {  
      messageType: 'M.1\_OWN\_UNIT',  
      picketAddressOctal: '0' \+ this.picketAddress.toString(8),  
      courseDeg: courseDeg,  
      speedKts: speedKts,  
      data24: data24,  
      ...encoded  
    };  
  }

  /\*\*  
   \* Encodes an M.2 Air Track Report Word  
   \* \[23:20\] Label (0x2 \= M.2)  
   \* \[19:17\] Identity (3 bits: Hostile, Friend, etc.)  
   \* \[16:7\]  Altitude (10 bits, 100-ft increments up to 102,300 ft)  
   \* \[6:0\]   Track Number Lower (7 bits)  
   \*/  
  encodeM2AirTrack(trackNum \= 42, identity \= TRACK\_IDENTITIES.HOSTILE, altitudeFt \= 25000\) {  
    const alt10 \= Math.min(1023, Math.round(altitudeFt / 100)) & 0x3FF;  
    const tn7 \= trackNum & 0x7F;

    let data24 \= 0;  
    data24 \|= (LINK11\_MESSAGE\_LABELS.M2\_AIR\_TRACK & 0x0F) \<\< 20;  
    data24 \|= (identity & 0x07) \<\< 17;  
    data24 \|= (alt10 & 0x3FF) \<\< 7;  
    data24 \|= (tn7 & 0x7F);

    const encoded \= Link11Engine.encodeWord30(data24);

    return {  
      messageType: 'M.2\_AIR\_TRACK',  
      trackNumber: trackNum,  
      identity: identity,  
      altitudeFt: alt10 \* 100,  
      data24: data24,  
      ...encoded  
    };  
  }

  /\*\*  
   \* Simulates a Net Control Station (NCS) Roll-Call Polling Cycle  
   \*/  
  executeRollCallStep() {  
    this.cycleCount++;  
    const targetPicket \= this.rollCallPickets\[this.currentPolledIndex\];  
    this.currentPolledIndex \= (this.currentPolledIndex \+ 1\) % this.rollCallPickets.length;

    // Call duration: Preamble (18ms) \+ Call word (13ms) \+ Picket response window (80ms)  
    const callDurationMs \= 111.0;  
    const fullCycleTimeSec \= (this.rollCallPickets.length \* callDurationMs) / 1000.0;

    return {  
      ncsCallSequence: this.cycleCount,  
      polledPicketOctal: '0' \+ targetPicket.toString(8),  
      callDurationMs: callDurationMs,  
      totalNetPickets: this.rollCallPickets.length,  
      netCycleTimeSec: fullCycleTimeSec.toFixed(2),  
      status: 'POLL\_ACKNOWLEDGED'  
    };  
  }  
}

if (typeof module \!== 'undefined' && module.exports) {  
  module.exports \= {  
    Link11Engine,  
    LINK11\_MESSAGE\_LABELS,  
    TRACK\_IDENTITIES,  
    WAVEFORM\_MODES  
  };  
}  
