/\*\*  
 \* TWAMPProbeForge Pro \- RFC 5357 TWAMP & TWAMP-Light Telemetry Engine  
 \* Encodes RFC 5357 Session-Sender and Session-Reflector test packets with 64-bit NTP timestamps,  
 \* computes microsecond-precision RTT, One-Way Delay (OWD), IPDV Jitter, and generates hex dumps.  
 \*/

(function (root, factory) {  
  if (typeof module \=== 'object' && module.exports) {  
    module.exports \= factory();  
  } else {  
    root.TWAMPEngine \= factory();  
  }  
})(typeof self \!== 'undefined' ? self : this, function () {  
  'use strict';

  // NTP epoch offset (1900-01-01 to 1970-01-01 in seconds)  
  const NTP\_OFFSET\_SEC \= 2208988800;

  function uint16ToBytes(val) {  
    const n \= Math.max(0, Math.min(0xffff, parseInt(val, 10\) || 0));  
    return \[(n \>\> 8\) & 0xff, n & 0xff\];  
  }

  function uint32ToBytes(val) {  
    const n \= Math.max(0, Math.min(0xffffffff, parseInt(val, 10\) || 0));  
    return \[  
      (n \>\>\> 24\) & 0xff,  
      (n \>\>\> 16\) & 0xff,  
      (n \>\>\> 8\) & 0xff,  
      n & 0xff  
    \];  
  }

  /\*\*  
   \* Converts a Unix millisecond timestamp to a 64-bit NTP timestamp (32-bit sec \+ 32-bit frac)  
   \*/  
  function unixMsToNtpBytes(unixMs) {  
    const totalSec \= (unixMs / 1000\) \+ NTP\_OFFSET\_SEC;  
    const sec \= Math.floor(totalSec);  
    const frac \= Math.floor((totalSec \- sec) \* 4294967296);  
    return \[...uint32ToBytes(sec), ...uint32ToBytes(frac)\];  
  }

  /\*\*  
   \* Encodes an RFC 5357 Session-Sender Test Packet (Unauthenticated Mode)  
   \* Sequence Number (4B) \+ Timestamp (8B) \+ Error Estimate (2B) \+ Padding  
   \*/  
  function encodeSenderPacket(options) {  
    const opts \= options || {};  
    const seq \= parseInt(opts.sequenceNumber || 1, 10);  
    const seqBytes \= uint32ToBytes(seq);  
    const tsBytes \= unixMsToNtpBytes(opts.sendTimestampMs || Date.now());  
      
    // Error Estimate: 2 bytes (bit 0=S, bit 1=Z, bits 2..7=Multiplier, bits 8..15=Scale)  
    const errorEstimate \= \[0x00, 0x01\]; // \~1 microsecond synchronized error estimate

    const paddingLen \= Math.max(0, parseInt(opts.paddingBytes || 26, 10));  
    const padding \= new Array(paddingLen).fill(0x5a); // 0x5a pattern

    return \[...seqBytes, ...tsBytes, ...errorEstimate, ...padding\];  
  }

  /\*\*  
   \* Encodes an RFC 5357 Session-Reflector Test Packet (Unauthenticated Mode)  
   \* 41 Bytes Header \+ Echoed Padding  
   \*/  
  function encodeReflectorPacket(options) {  
    const opts \= options || {};  
    const reflectorSeq \= uint32ToBytes(parseInt(opts.reflectorSequenceNumber || 1, 10));  
    const recvTs \= unixMsToNtpBytes(opts.receiveTimestampMs || (Date.now() \+ 2)); // T2  
    const errEst \= \[0x00, 0x01\];  
    const mbz1 \= \[0x00, 0x00\];  
    const transTs \= unixMsToNtpBytes(opts.transmitTimestampMs || (Date.now() \+ 3)); // T3  
    const senderSeq \= uint32ToBytes(parseInt(opts.senderSequenceNumber || 1, 10));  
    const senderTs \= unixMsToNtpBytes(opts.senderTimestampMs || Date.now()); // T1  
    const senderErrEst \= \[0x00, 0x01\];  
    const mbz2 \= \[0x00, 0x00\];  
    const senderTtl \= \[Math.max(1, Math.min(255, parseInt(opts.senderTtl || 64, 10)))\];

    const paddingLen \= Math.max(0, parseInt(opts.paddingBytes || 26, 10));  
    const padding \= new Array(paddingLen).fill(0x5a);

    return \[  
      ...reflectorSeq,  
      ...recvTs,  
      ...errEst,  
      ...mbz1,  
      ...transTs,  
      ...senderSeq,  
      ...senderTs,  
      ...senderErrEst,  
      ...mbz2,  
      ...senderTtl,  
      ...padding  
    \];  
  }

  /\*\*  
   \* Computes precise Two-Way RTT, One-Way Forward Delay, One-Way Reverse Delay & Jitter  
   \*/  
  function computeSlaMetrics(t1Ms, t2Ms, t3Ms, t4Ms, prevFwdDelayMs) {  
    // RTT \= (T4 \- T1) \- (T3 \- T2)  
    const rawRtt \= (t4Ms \- t1Ms);  
    const reflectorProcessingTime \= (t3Ms \- t2Ms);  
    const netRtt \= Math.max(0, rawRtt \- reflectorProcessingTime);

    const fwdDelay \= (t2Ms \- t1Ms);  
    const revDelay \= (t4Ms \- t3Ms);

    let ipdvJitter \= 0;  
    if (prevFwdDelayMs \!== undefined && prevFwdDelayMs \!== null) {  
      ipdvJitter \= Math.abs(fwdDelay \- prevFwdDelayMs);  
    }

    return {  
      roundTripDelayMs: parseFloat(netRtt.toFixed(3)),  
      forwardDelayMs: parseFloat(fwdDelay.toFixed(3)),  
      reverseDelayMs: parseFloat(revDelay.toFixed(3)),  
      reflectorDwellMs: parseFloat(reflectorProcessingTime.toFixed(3)),  
      jitterIpdvMs: parseFloat(ipdvJitter.toFixed(3))  
    };  
  }

  /\*\*  
   \* Formats byte array into Wireshark-style formatted hex dump  
   \*/  
  function formatHexDump(byteArray) {  
    if (\!byteArray || \!byteArray.length) return '';  
    const lines \= \[\];  
    const bytesPerLine \= 16;

    for (let i \= 0; i \< byteArray.length; i \+= bytesPerLine) {  
      const slice \= byteArray.slice(i, i \+ bytesPerLine);  
      const offset \= i.toString(16).padStart(4, '0');  
        
      const hexPart \= slice  
        .map(b \=\> b.toString(16).padStart(2, '0'))  
        .join(' ');  
      const paddedHex \= hexPart.padEnd(bytesPerLine \* 3, ' ');

      const asciiPart \= slice  
        .map(b \=\> (b \>= 32 && b \<= 126 ? String.fromCharCode(b) : '.'))  
        .join('');

      lines.push(\`\${offset}:  \${paddedHex} |\${asciiPart}|\`);  
    }

    return lines.join('\\n');  
  }

  return {  
    encodeSenderPacket,  
    encodeReflectorPacket,  
    computeSlaMetrics,  
    formatHexDump  
  };  
});  
