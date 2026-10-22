/\*\*  
 \* TWAMPProbeForge Pro \- Application State Orchestrator  
 \*/

document.addEventListener('DOMContentLoaded', () \=\> {  
  const PRESETS \= {  
    '5g-urllc-slice-sla-cisco': {  
      senderName: '5G\_UPF\_PROBE\_SENDER',  
      senderIp: '198.51.100.1',  
      reflectorIp: '198.51.100.2',  
      senderPort: 862,  
      reflectorPort: 862,  
      clockSyncType: 'HARDWARE\_PTP',  
      authMode: 'AUTHENTICATED',  
      senderAclEnabled: true,  
      dscpValue: 46,  
      paddingBytes: 1400,  
      reflectorRateLimitPps: 500,  
      oobVrfEnabled: true,  
      vrfName: 'sla-telemetry',  
      probeIntervalMs: 100,  
      rttThresholdMs: 5,  
      jitterThresholdMs: 1,  
      lossThresholdPct: 0.01,  
      hardwareTimestamping: true,  
      validateAsymmetry: true  
    },  
    'metro-financial-low-latency-junos': {  
      senderName: 'FINTECH\_LOW\_LATENCY\_TEST',  
      senderIp: '198.51.100.10',  
      reflectorIp: '198.51.100.20',  
      senderPort: 862,  
      reflectorPort: 862,  
      clockSyncType: 'PTP\_1588',  
      authMode: 'AUTHENTICATED',  
      senderAclEnabled: true,  
      dscpValue: 48,  
      paddingBytes: 512,  
      reflectorRateLimitPps: 200,  
      oobVrfEnabled: true,  
      vrfName: 'fin-telemetry',  
      probeIntervalMs: 250,  
      rttThresholdMs: 2,  
      jitterThresholdMs: 0.5,  
      lossThresholdPct: 0.001,  
      hardwareTimestamping: true,  
      validateAsymmetry: true  
    },  
    'vulnerable-unauthenticated-reflector': {  
      senderName: 'FRAGILE\_PROBE',  
      senderIp: '203.0.113.1',  
      reflectorIp: '203.0.113.2',  
      senderPort: 862,  
      reflectorPort: 862,  
      clockSyncType: 'UNVERIFIED',  
      authMode: 'NONE',  
      senderAclEnabled: false,  
      dscpValue: 0,  
      paddingBytes: 0,  
      reflectorRateLimitPps: 0,  
      oobVrfEnabled: false,  
      vrfName: 'default',  
      probeIntervalMs: 5000,  
      rttThresholdMs: 0,  
      jitterThresholdMs: 0,  
      lossThresholdPct: 0,  
      hardwareTimestamping: false,  
      validateAsymmetry: false  
    }  
  };

  let activeFormat \= 'cisco';

  function getFormData() {  
    return {  
      senderName: document.getElementById('senderName').value,  
      senderIp: document.getElementById('senderIp').value,  
      reflectorIp: document.getElementById('reflectorIp').value,  
      senderPort: parseInt(document.getElementById('senderPort').value, 10\) || 862,  
      reflectorPort: parseInt(document.getElementById('reflectorPort').value, 10\) || 862,  
      clockSyncType: document.getElementById('clockSyncType').value,  
      authMode: document.getElementById('senderAclEnabled').checked ? 'AUTHENTICATED' : 'NONE',  
      senderAclEnabled: document.getElementById('senderAclEnabled').checked,  
      dscpValue: parseInt(document.getElementById('dscpValue').value, 10\) || 0,  
      paddingBytes: parseInt(document.getElementById('paddingBytes').value, 10\) || 0,  
      probeIntervalMs: parseInt(document.getElementById('probeIntervalMs').value, 10\) || 100,  
      reflectorRateLimitPps: parseInt(document.getElementById('reflectorRateLimitPps').value, 10\) || 0,  
      rttThresholdMs: parseFloat(document.getElementById('rttThresholdMs').value) || 0,  
      jitterThresholdMs: parseFloat(document.getElementById('jitterThresholdMs').value) || 0,  
      lossThresholdPct: 0.01,  
      hardwareTimestamping: document.getElementById('hardwareTimestamping').checked,  
      validateAsymmetry: document.getElementById('validateAsymmetry').checked,  
      oobVrfEnabled: document.getElementById('oobVrfEnabled').checked,  
      vrfName: document.getElementById('oobVrfEnabled').checked ? 'sla-telemetry' : 'default'  
    };  
  }

  function applyDataToForm(data) {  
    if (\!data) return;  
    document.getElementById('senderName').value \= data.senderName || '';  
    document.getElementById('senderIp').value \= data.senderIp || '198.51.100.1';  
    document.getElementById('reflectorIp').value \= data.reflectorIp || '198.51.100.2';  
    document.getElementById('senderPort').value \= data.senderPort || 862;  
    document.getElementById('reflectorPort').value \= data.reflectorPort || 862;  
    document.getElementById('clockSyncType').value \= data.clockSyncType || 'UNVERIFIED';  
    document.getElementById('dscpValue').value \= data.dscpValue \!== undefined ? data.dscpValue : 46;  
    document.getElementById('paddingBytes').value \= data.paddingBytes \!== undefined ? data.paddingBytes : 1400;  
    document.getElementById('probeIntervalMs').value \= data.probeIntervalMs || 100;  
    document.getElementById('reflectorRateLimitPps').value \= data.reflectorRateLimitPps || 500;  
    document.getElementById('rttThresholdMs').value \= data.rttThresholdMs || 5;  
    document.getElementById('jitterThresholdMs').value \= data.jitterThresholdMs || 1;  
    document.getElementById('senderAclEnabled').checked \= \!\!data.senderAclEnabled;  
    document.getElementById('hardwareTimestamping').checked \= \!\!data.hardwareTimestamping;  
    document.getElementById('validateAsymmetry').checked \= \!\!data.validateAsymmetry;  
    document.getElementById('oobVrfEnabled').checked \= \!\!data.oobVrfEnabled;  
    renderAll();  
  }

  function renderAudit(data) {  
    const auditRes \= TWAMPSecurityAuditor.auditConfiguration(data);  
    const scoreBadge \= document.getElementById('scoreBadge');  
    const headerScoreBadge \= document.getElementById('headerScoreBadge');  
    const scoreText \= document.getElementById('scoreText');  
    const scoreSummary \= document.getElementById('scoreSummary');  
    const scoreBanner \= document.getElementById('scoreBanner');  
    const auditList \= document.getElementById('auditList');

    scoreBadge.textContent \= auditRes.grade;  
    headerScoreBadge.textContent \= \`SCORE: \${auditRes.score}/100 (\${auditRes.grade})\`;  
    scoreText.textContent \= \`Zero-Trust SLA Score: \${auditRes.score} / 100\`;

    if (auditRes.grade \=== 'A+' || auditRes.grade \=== 'A') {  
      scoreBadge.style.color \= '\#10b981';  
      scoreBadge.style.borderColor \= '\#10b981';  
      scoreBadge.style.background \= 'rgba(16, 185, 129, 0.2)';  
      scoreBanner.style.background \= 'rgba(16, 185, 129, 0.1)';  
      scoreBanner.style.borderColor \= 'rgba(16, 185, 129, 0.3)';  
      scoreSummary.textContent \= 'Hardware PTP Sync • Reflector ACL Guard • DSCP EF Tagging • 1400B Frame Sizing';  
    } else if (auditRes.grade \=== 'B' || auditRes.grade \=== 'C') {  
      scoreBadge.style.color \= '\#f59e0b';  
      scoreBadge.style.borderColor \= '\#f59e0b';  
      scoreBadge.style.background \= 'rgba(245, 158, 11, 0.2)';  
      scoreBanner.style.background \= 'rgba(245, 158, 11, 0.1)';  
      scoreBanner.style.borderColor \= 'rgba(245, 158, 11, 0.3)';  
      scoreSummary.textContent \= 'Warning: Sub-millisecond NTP or software timestamping may slightly skew micro-jitter accuracy.';  
    } else {  
      scoreBadge.style.color \= '\#ef4444';  
      scoreBadge.style.borderColor \= '\#ef4444';  
      scoreBadge.style.background \= 'rgba(239, 68, 68, 0.2)';  
      scoreBanner.style.background \= 'rgba(239, 68, 68, 0.1)';  
      scoreBanner.style.borderColor \= 'rgba(239, 68, 68, 0.3)';  
      scoreSummary.textContent \= 'CRITICAL: Open unauthenticated reflector (DDoS amplifier), unverified clock, and unbounded CPU rate.';  
    }

    auditList.innerHTML \= '';  
    auditRes.findings.forEach(f \=\> {  
      const item \= document.createElement('div');  
      item.className \= \`audit-item \${f.status}\`;  
      item.innerHTML \= \`  
        \<div\>  
          \<strong\>\${f.title}\</strong\>  
          \<p style="font-size: 0.75rem; color: \#94a3b8; margin-top: 2px;"\>\${f.desc}\</p\>  
        \</div\>  
        \<span style="font-weight: 700; font-family: monospace;"\>\${f.status} (\${f.points} pts)\</span\>  
      \`;  
      auditList.appendChild(item);  
    });  
  }

  function renderHex(data) {  
    const pktBytes \= TWAMPEngine.encodeSenderPacket({  
      sequenceNumber: 1,  
      sendTimestampMs: Date.now(),  
      paddingBytes: data.paddingBytes  
    });  
    const hexBox \= document.getElementById('hexBox');  
    const dump \= TWAMPEngine.formatHexDump(pktBytes);

    hexBox.textContent \= \`// RFC 5357 TWAMP-Test Session-Sender Packet (\${pktBytes.length} Bytes):\\n\` \+ dump;  
  }

  function renderConfig(data) {  
    const configOutput \= document.getElementById('configOutput');  
    let code \= '';

    switch (activeFormat) {  
      case 'cisco':  
        code \= TWAMPConfigSynthesizer.synthesizeCisco(data);  
        break;  
      case 'junos':  
        code \= TWAMPConfigSynthesizer.synthesizeJunos(data);  
        break;  
      case 'nokia':  
        code \= TWAMPConfigSynthesizer.synthesizeNokia(data);  
        break;  
      case 'linux':  
        code \= TWAMPConfigSynthesizer.synthesizeLinuxReflector(data);  
        break;  
      case 'prom':  
        code \= TWAMPConfigSynthesizer.synthesizePrometheusExporter(data);  
        break;  
      default:  
        code \= '\! Select a format';  
    }

    configOutput.textContent \= code;  
  }

  function renderAll() {  
    const data \= getFormData();  
    renderAudit(data);  
    renderHex(data);  
    renderConfig(data);  
  }

  // Event Listeners  
  document.getElementById('presetSelect').addEventListener('change', (e) \=\> {  
    const presetKey \= e.target.value;  
    if (PRESETS\[presetKey\]) {  
      applyDataToForm(PRESETS\[presetKey\]);  
    }  
  });

  document.getElementById('btnResetPreset').addEventListener('click', () \=\> {  
    const presetKey \= document.getElementById('presetSelect').value;  
    if (PRESETS\[presetKey\]) {  
      applyDataToForm(PRESETS\[presetKey\]);  
    }  
  });

  document.getElementById('btnRecalculate').addEventListener('click', renderAll);

  // Form Inputs live re-render  
  document.querySelectorAll('input, select').forEach(elem \=\> {  
    elem.addEventListener('input', renderAll);  
    elem.addEventListener('change', renderAll);  
  });

  // Synthesizer Tabs  
  document.querySelectorAll('.tab-btn').forEach(btn \=\> {  
    btn.addEventListener('click', () \=\> {  
      document.querySelectorAll('.tab-btn').forEach(b \=\> b.classList.remove('active'));  
      btn.classList.add('active');  
      activeFormat \= btn.getAttribute('data-format');  
      renderConfig(getFormData());  
    });  
  });

  // Copy Buttons  
  document.getElementById('btnCopyHex').addEventListener('click', () \=\> {  
    const hex \= document.getElementById('hexBox').textContent;  
    navigator.clipboard.writeText(hex).then(() \=\> {  
      alert('TWAMP-Test packet hex dump copied to clipboard\!');  
    });  
  });

  document.getElementById('btnCopyConfig').addEventListener('click', () \=\> {  
    const cfg \= document.getElementById('configOutput').textContent;  
    navigator.clipboard.writeText(cfg).then(() \=\> {  
      alert('Synthesized TWAMP configuration copied to clipboard\!');  
    });  
  });

  // Initial render  
  renderAll();  
});  
