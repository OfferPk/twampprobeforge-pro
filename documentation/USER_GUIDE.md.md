\# TWAMPProbeForge Pro \- User Guide & CI/CD Telemetry Pipeline Manual

\#\# 1\. Web Studio Quickstart

1\. Open \`source-code/index.html\` in any modern web browser.  
2\. Select an industry preset from the top dropdown:  
   \- \*\*5G URLLC Slice SLA Cisco (Grade A+)\*\*: 5G UPF to gNodeB active measurement session using hardware PTP (sub-microsecond sync), DSCP EF 46, 1400B frame padding, 100ms interval, and strict reflector ACL whitelisting.  
   \- \*\*Metro Financial Low-Latency Junos (Grade A+)\*\*: High-frequency trading transport ring probe using boundary clock PTP, DSCP CS6 48, and 2ms RTT SLA alert threshold.  
   \- \*\*Hazardous Unauthenticated Reflector (Grade F)\*\*: Demonstrates failure modes of open unauthenticated reflectors (DDoS amplification risk), unverified clocks, 0B padding, and Best-Effort queueing.  
3\. Review the \*\*10-Point Zero-Trust Scorecard\*\* to inspect any clock skew, reflection attack, or CPU exhaustion risks.  
4\. Modify probe parameters (Sender/Reflector IP, Port, DSCP, Padding, Thresholds).  
5\. Inspect the generated \*\*RFC 5357 TWAMP-Test Packet Hex Dump\*\* with byte offsets and 64-bit NTP timestamps.  
6\. Switch synthesizer tabs to export production configuration:  
   \- \*\*Cisco IOS-XR\*\*: \`ipsla\`, \`type twamp-light\`, \`responder twamp-light\`.  
   \- \*\*Juniper Junos\*\*: \`set services rpm probe\`, \`server-reflector\`.  
   \- \*\*Nokia SR-OS\*\*: \`/configure oam-pm twamp-light\`.  
   \- \*\*Linux Reflector Daemon\*\*: Standalone Python UDP reflector script.  
   \- \*\*Prometheus Exporter\*\*: Prometheus metrics endpoint on \`:9180/metrics\`.

\---

\#\# 2\. DevSecOps CLI Tool (\`twampprobeforge-cli.py\`)

The standalone Python CLI tool requires zero external dependencies and runs on Python 3.8+.

\#\#\# 2.1 Encoding TWAMP-Test Packet to Binary Hex  
\`\`\`bash  
python3 build/twampprobeforge-cli.py encode-packet \-p presets/5g-urllc-slice-sla-cisco.json \--role sender  
\`\`\`

\#\#\# 2.2 Computing Microsecond SLA Metrics  
\`\`\`bash  
python3 build/twampprobeforge-cli.py calc-sla \--t1 1000.0 \--t2 1002.5 \--t3 1002.7 \--t4 1005.4  
\`\`\`  
Output:  
\`\`\`  
\=== TWAMP Microsecond SLA Computation \===  
Round-Trip Delay (RTT): 5.2 ms  
One-Way Forward Delay:  2.5 ms  
One-Way Reverse Delay:  2.7 ms  
Reflector Dwell Time:   0.2 ms  
\`\`\`

\#\#\# 2.3 Running CI/CD Pipeline Telemetry Audits  
\`\`\`bash  
\# Clean production configuration \-\> Exits code 0  
python3 build/twampprobeforge-cli.py audit \-p presets/5g-urllc-slice-sla-cisco.json \--strict

\# Insecure/fragile configuration \-\> Exits code 1 (Pipeline Fails)  
python3 build/twampprobeforge-cli.py audit \-p presets/vulnerable-unauthenticated-reflector.json \--strict  
\`\`\`

\#\#\# 2.4 Synthesizing Router Configurations  
\`\`\`bash  
python3 build/twampprobeforge-cli.py synthesize \-p presets/5g-urllc-slice-sla-cisco.json \-f cisco  
python3 build/twampprobeforge-cli.py synthesize \-p presets/metro-financial-low-latency-junos.json \-f junos  
\`\`\`

\---

\#\# 3\. GitHub Actions CI/CD Pipeline Gate

Add this workflow to your NetDevOps repository (\`.github/workflows/twamp-sla-gate.yml\`):

\`\`\`yaml  
name: Zero-Trust TWAMP SLA CI/CD Gate

on:  
  pull\_request:  
    paths:  
      \- 'twamp/\*\*'  
      \- 'telemetry/\*\*'  
      \- 'sla/\*\*'

jobs:  
  twamp-audit:  
    runs-on: ubuntu-latest  
    steps:  
      \- name: Checkout Code  
        uses: actions/checkout@v4

      \- name: Set up Python  
        uses: actions/setup-python@v5  
        with:  
          python-version: '3.11'

      \- name: Audit TWAMP SLA Session (Strict Gate)  
        run: |  
          python3 build/twampprobeforge-cli.py audit \-p presets/5g-urllc-slice-sla-cisco.json \--strict  
\`\`\`  
