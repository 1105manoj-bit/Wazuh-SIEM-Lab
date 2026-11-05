# Wazuh SIEM Home Lab Build, Breaking It, and Fixing the Detection Gaps

Stack: Wazuh 4.14, Kali Linux, Ubuntu, Windows, Hydra, Apache Bench, MITRE ATT&CK

I set this up to actually learn how a SOC analyst's tools work day to day, not just read about them. It's a home lab three VMs on one laptop, not real infrastructure, and I'd rather just say that upfront than have someone assume otherwise. The troubleshooting along the way ended up teaching me more than the install itself, so that's most of what's below.

## Setup

- **Ubuntu VM** this is where the Wazuh manager, indexer, and dashboard actually live. The real SIEM.
- **Kali Linux** runs a wazuh-agent. Does double duty here: it's a monitored endpoint, and it's also the box I run attacks *from*, so I can watch my own traffic land as an alert over on the Ubuntu dashboard.
- **Windows VM** separate wazuh-agent, another monitored endpoint.

Each VM has two network interfaces: a NAT adapter and a VirtualBox Host-only adapter. That second one mattered way more than I expected more on that below.

## Stuff that broke, and why

**wazuh-agent wouldn't install.** I'd initially set Kali up as an all-in-one Wazuh box manager, indexer, dashboard, all on one VM and then tried installing wazuh-agent on top of it to test enrollment. Hard conflict at the package level; turns out you can't run wazuh-agent and wazuh-manager on the same host at all. A manager already watches itself anyway (shows up as agent ID `000` in the agent list), so there's no real need to bolt a separate agent on top of it either.

**Agent kept trying to enroll and failing, for hours.** After removing the manager from Kali to get the agent installed, the dashboard's old "Deploy new agent" wizard had auto-filled a server address for it to enroll to. I assumed that address pointed at a different machine. It didn't `ip a` showed it was literally Kali's own second NIC, left over from when Kali was still the manager. Classic case of debugging a "network problem" that isn't actually a network problem: the host answered fine, there was just nothing listening on that port anymore, since the manager I was trying to reach was the one I'd just removed. That's the point where moving the manager to its own dedicated box the Ubuntu VM stopped being optional and just became the obvious fix.

**Dashboard went offline after reinstalling the manager.** Fresh manager install means fresh random credentials, which no longer matched what the dashboard had saved from before. Fixed with Wazuh's own credential sync tool (`wazuh-passwords-tool.sh --change`), restarted the dashboard, moved on.

## Actually attacking it

SSH brute force first, since it's the easy one:

```bash
hydra -l notarealuser -P /usr/share/wordlists/rockyou.txt ssh://127.0.*.* -t 4
```

This one just worked. Default Wazuh rules catch repeated SSH failures fine with zero extra config shows up as a brute-force alert in the dashboard, maps to MITRE ATT&CK T1110.

Then I tried an HTTP flood with Apache Bench:

```bash
ab -n 5000 -c 50 http://127.*.*.1/
```

Ran clean, zero failed requests on the Apache side. And then... nothing showed up in Wazuh. At all. Which was annoying for a good twenty minutes until I realized it wasn't one problem, it was two stacked on top of each other:

1. Wazuh doesn't read Apache's access log by default you have to tell it to.
2. Even once it does, there's no default rule for "a lot of requests really fast." A wall of 200 OKs just doesn't look suspicious to the out-of-the-box ruleset.

So, log source added on the agent side:

```xml
<localfile>
  <log_format>apache</log_format>
  <location>/var/log/apache2/access.log</location>
</localfile>
```

And a rule written for it on the manager side:

```xml
<group name="local,web_flood,">
  <rule id="100100" level="10" frequency="50" timeframe="2">
    <if_group>web-accesslog</if_group>
    <same_source_ip />
    <description>Possible web flood/DoS: 50+ requests from the same IP within 2 seconds.</description>
    <group>web_flood,</group>
  </rule>
</group>
```

Ran the flood again. Got the alert this time, rule `100100`, correct source IP attached to it.

## Tools

Wazuh, Hydra, Apache Bench, Apache2, nmap/nc for network debugging, MITRE ATT&CK (Wazuh's built-in mapping view)

## What I'd add next

- Suricata, since Wazuh alone can't really see true network-layer floods only what shows up in logs
- Atomic Red Team, for broader technique coverage beyond just brute force and flood
- A genuinely vulnerable target (Metasploitable2), so I can test detection against a real exploit chain instead of just noisy traffic