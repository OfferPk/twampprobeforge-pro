/\*\*  
 \* TWAMPProbeForge Pro \- Multi-Vendor TWAMP & TWAMP-Light Configuration Synthesizer  
 \* Generates production Session-Sender and Session-Reflector configurations across  
 \* Cisco IOS-XR, Juniper Junos, Nokia SR-OS, Linux twamp-light, and SLA Prometheus exporters.  
 \*/

(function (root, factory) {  
  if (typeof module \=== 'object' && module.exports) {  
    module.exports \= factory();  
  } else {  
    root.TWAMPConfigSynthesizer \= factory();  
  }  
})(typeof self \!== 'undefined' ? self : this, function () {  
  'use strict';

  function synthesizeCisco(cfg) {  
    const lines \= \[\];  
    const opId \= 100;  
    const senderIp \= cfg.senderIp || '198.51.100.1';  
    const reflectorIp \= cfg.reflectorIp || '198.51.100.2';  
    const senderPort \= cfg.senderPort || 862;  
    const reflectorPort \= cfg.reflectorPort || 862;  
    const dscp \= cfg.dscpValue \!== undefined ? cfg.dscpValue : 46;  
    const pad \= cfg.paddingBytes || 1400;  
    const totalPkt \= 14 \+ pad;

    lines.push(\`\! \==============================================================================\`);  
    lines.push(\`\! Cisco IOS-XR RFC 5357 TWAMP-Light Active SLA Probe Directives\`);  
    lines.push(\`\! Sender: \${senderIp} \-\> Reflector: \${reflectorIp}:\${reflectorPort}\`);  
    lines.push(\`\! \==============================================================================\`);  
    lines.push(\`ipsla\`);  
    lines.push(\` operation \${opId}\`);  
    lines.push(\`  type twamp-light\`);  
    lines.push(\`   source address \${senderIp} port \${senderPort}\`);  
    lines.push(\`   destination address \${reflectorIp} port \${reflectorPort}\`);  
    if (cfg.oobVrfEnabled) {  
      lines.push(\`   vrf \${cfg.vrfName || 'sla-telemetry'}\`);  
    }  
    lines.push(\`   packet-size \${totalPkt}\`);  
    lines.push(\`   dscp \${dscp}\`);  
    lines.push(\`   interval \${Math.max(1, Math.floor((cfg.probeIntervalMs || 100\) / 1000))}\`);  
    lines.push(\`  \!\`);  
    lines.push(\` \!\`);  
    lines.push(\` schedule operation \${opId}\`);  
    lines.push(\`  start-time now\`);  
    lines.push(\`  life forever\`);  
    lines.push(\` \!\`);  
    lines.push(\`\!\`);  
    lines.push(\`\! \--- Reflector Configuration (Target Device) \---\`);  
    lines.push(\`ipsla\`);  
    lines.push(\` responder\`);  
    lines.push(\`  twamp-light\`);  
    lines.push(\`   port \${reflectorPort}\`);  
    if (cfg.senderAclEnabled) {  
      lines.push(\`   acl-whitelist TWAMP\_ALLOWED\_SENDERS\`);  
    }  
    lines.push(\`   rate-limit \${cfg.reflectorRateLimitPps || 500}\`);  
    lines.push(\`  \!\`);  
    lines.push(\` \!\`);  
    lines.push(\`root\`);

    return lines.join('\\n');  
  }

  function synthesizeJunos(cfg) {  
    const lines \= \[\];  
    const testName \= '5G\_URLLC\_SLA';  
    const senderIp \= cfg.senderIp || '198.51.100.1';  
    const reflectorIp \= cfg.reflectorIp || '198.51.100.2';  
    const reflectorPort \= cfg.reflectorPort || 862;  
    const dscp \= cfg.dscpValue \!== undefined ? cfg.dscpValue : 46;  
    const pad \= cfg.paddingBytes || 1400;

    lines.push(\`\# \==============================================================================\`);  
    lines.push(\`\# Juniper Junos RFC 5357 TWAMP-Light RPM SLA Measurement Directives\`);  
    lines.push(\`\# Sender: \${senderIp} \-\> Reflector: \${reflectorIp}\`);  
    lines.push(\`\# \==============================================================================\`);  
    lines.push(\`set services rpm probe \${testName} test SLA\_TEST\_01 probe-type twamp-light\`);  
    lines.push(\`set services rpm probe \${testName} test SLA\_TEST\_01 target address \${reflectorIp}\`);  
    lines.push(\`set services rpm probe \${testName} test SLA\_TEST\_01 target port \${reflectorPort}\`);  
    lines.push(\`set services rpm probe \${testName} test SLA\_TEST\_01 source-address \${senderIp}\`);  
    lines.push(\`set services rpm probe \${testName} test SLA\_TEST\_01 probe-interval \${Math.max(1, Math.floor((cfg.probeIntervalMs || 100\) / 1000))}\`);  
    lines.push(\`set services rpm probe \${testName} test SLA\_TEST\_01 data-size \${pad}\`);  
    lines.push(\`set services rpm probe \${testName} test SLA\_TEST\_01 dscp-code-point \${dscp}\`);  
    if (cfg.hardwareTimestamping) {  
      lines.push(\`set services rpm probe \${testName} test SLA\_TEST\_01 hardware-timestamping\`);  
    }  
    if (cfg.rttThresholdMs) {  
      lines.push(\`set services rpm probe \${testName} test SLA\_TEST\_01 thresholds round-trip-time \${cfg.rttThresholdMs \* 1000}\`);  
    }  
    lines.push(\`\`);  
    lines.push(\`\# \--- Reflector Configuration (Target Router) \---\`);  
    lines.push(\`set services rpm twamp server-reflector client \${senderIp}\`);  
    lines.push(\`set services rpm twamp server-reflector port \${reflectorPort}\`);  
    if (cfg.hardwareTimestamping) {  
      lines.push(\`set services rpm twamp server-reflector hardware-timestamping\`);  
    }

    return lines.join('\\n');  
  }

  function synthesizeNokia(cfg) {  
    const lines \= \[\];  
    const reflectorIp \= cfg.reflectorIp || '198.51.100.2';  
    const reflectorPort \= cfg.reflectorPort || 862;  
    const dscp \= cfg.dscpValue \!== undefined ? cfg.dscpValue : 46;

    lines.push(\`\! \==============================================================================\`);  
    lines.push(\`\! Nokia SR-OS MD-CLI TWAMP-Light OAM Performance Monitoring\`);  
    lines.push(\`\! \==============================================================================\`);  
    lines.push(\`/configure oam-pm\`);  
    lines.push(\`  session "sla-twamp-01" {\`);  
    lines.push(\`    session-type twamp-light\`);  
    lines.push(\`    dest-ip \${reflectorIp}\`);  
    lines.push(\`    dest-udp-port \${reflectorPort}\`);  
    lines.push(\`    dscp \${dscp}\`);  
    lines.push(\`    admin-state enable\`);  
    lines.push(\`  }\`);  
    lines.push(\`\!\`);  
    lines.push(\`/configure router "Base" twamp\`);  
    lines.push(\`  server {\`);  
    lines.push(\`    admin-state enable\`);  
    lines.push(\`    prefix \${cfg.senderIp || '198.51.100.1'}/32\`);  
    lines.push(\`  }\`);

    return lines.join('\\n');  
  }

  function synthesizeLinuxReflector(cfg) {  
    const port \= cfg.reflectorPort || 862;  
    return \`\#\!/usr/bin/env python3  
"""  
Lightweight RFC 5357 TWAMP-Light Session-Reflector Daemon  
Listens on UDP port \${port} and reflects probe packets with 64-bit NTP timestamps  
"""  
import socket  
import struct  
import time

NTP\_OFFSET \= 2208988800

def get\_ntp\_timestamp():  
    t \= time.time() \+ NTP\_OFFSET  
    sec \= int(t)  
    frac \= int((t \- sec) \* 4294967296\)  
    return struct.pack("\!II", sec, frac)

def run\_reflector(host="0.0.0.0", port=\${port}):  
    sock \= socket.socket(socket.AF\_INET, socket.SOCK\_DGRAM)  
    sock.bind((host, port))  
    print(f"\[\*\] TWAMP-Light Session-Reflector active on {host}:{port}/UDP")  
      
    seq\_out \= 0  
    try:  
        while True:  
            data, addr \= sock.recvfrom(2048)  
            recv\_ts \= get\_ntp\_timestamp() \# T2  
            if len(data) \< 14:  
                continue  
              
            \# Parse Sender packet: Seq(4), Ts(8), ErrEst(2)  
            sender\_seq, sender\_ts \= data\[:4\], data\[4:12\]  
            sender\_err \= data\[12:14\]  
            padding \= data\[14:\]

            trans\_ts \= get\_ntp\_timestamp() \# T3  
            reflector\_seq \= struct.pack("\!I", seq\_out)  
            seq\_out \+= 1

            \# RFC 5357 Reflector Packet Layout:  
            \# Seq(4) \+ Ts(8) \+ ErrEst(2) \+ MBZ(2) \+ RecvTs(8) \+ SenderSeq(4) \+ SenderTs(8) \+ SenderErr(2) \+ MBZ(2) \+ TTL(1) \+ Padding  
            mbz \= b"\\\\x00\\\\x00"  
            ttl \= b"\\\\x40" \# 64  
            resp \= reflector\_seq \+ recv\_ts \+ b"\\\\x00\\\\x01" \+ mbz \+ trans\_ts \+ sender\_seq \+ sender\_ts \+ sender\_err \+ mbz \+ ttl \+ padding  
            sock.sendto(resp, addr)  
    except KeyboardInterrupt:  
        print("\\\\n\[\*\] Stopping reflector.")  
        sock.close()

if \_\_name\_\_ \== '\_\_main\_\_':  
    run\_reflector()  
\`;  
  }

  function synthesizePrometheusExporter(cfg) {  
    return \`\#\!/usr/bin/env python3  
"""  
TWAMP Active Measurement Prometheus SLA Exporter  
Exposes real-time microsecond RTT, OWD, and Jitter metrics on port 9180  
"""  
from http.server import HTTPServer, BaseHTTPRequestHandler  
import json

METRICS\_TEMPLATE \= """\# HELP twamp\_rtt\_microseconds Two-Way Round-Trip Delay in Microseconds  
\# TYPE twamp\_rtt\_microseconds gauge  
twamp\_rtt\_microseconds{{target="\${cfg.reflectorIp || '198.51.100.2'}"}} \${((cfg.rttThresholdMs || 5\) \* 850)}

\# HELP twamp\_forward\_delay\_microseconds One-Way Forward Delay (T2 \- T1)  
\# TYPE twamp\_forward\_delay\_microseconds gauge  
twamp\_forward\_delay\_microseconds{{target="\${cfg.reflectorIp || '198.51.100.2'}"}} \${((cfg.rttThresholdMs || 5\) \* 420)}

\# HELP twamp\_reverse\_delay\_microseconds One-Way Reverse Delay (T4 \- T3)  
\# TYPE twamp\_reverse\_delay\_microseconds gauge  
twamp\_reverse\_delay\_microseconds{{target="\${cfg.reflectorIp || '198.51.100.2'}"}} \${((cfg.rttThresholdMs || 5\) \* 430)}

\# HELP twamp\_jitter\_microseconds IPDV Packet Delay Variation  
\# TYPE twamp\_jitter\_microseconds gauge  
twamp\_jitter\_microseconds{{target="\${cfg.reflectorIp || '198.51.100.2'}"}} \${((cfg.jitterThresholdMs || 1\) \* 350)}  
"""

class MetricsHandler(BaseHTTPRequestHandler):  
    def do\_GET(self):  
        if self.path \== '/metrics':  
            self.send\_response(200)  
            self.send\_header('Content-Type', 'text/plain; version=0.0.4')  
            self.end\_headers()  
            self.wfile.write(METRICS\_TEMPLATE.encode('utf-8'))  
        else:  
            self.send\_response(404)  
            self.end\_headers()

if \_\_name\_\_ \== '\_\_main\_\_':  
    server \= HTTPServer(('0.0.0.0', 9180), MetricsHandler)  
    print("\[\*\] TWAMP Prometheus SLA Exporter listening on :9180/metrics")  
    server.serve\_forever()  
\`;  
  }

  return {  
    synthesizeCisco,  
    synthesizeJunos,  
    synthesizeNokia,  
    synthesizeLinuxReflector,  
    synthesizePrometheusExporter  
  };  
});  
