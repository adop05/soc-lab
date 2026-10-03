Phase 1: Infrastructure Provisioning & Environment Setup
Provisioned Fedora (soc) machine using Proxmox Automation pipeline.
Installed Docker and Docker Compose.
Created directory structure:
"Mermaid Tree showcasing folder structure inside soc-lab directory"

Phase 2: docker-compose.yml

`services:
    suricata:
        image: jasonish/suricata:latest
        container_name: suricata
        network_mode: "host"
        cap_add:
            - NET_ADMIN
            - NET_RAW
            - SYS_NICE
        volumes:
            - ./suricata/logs:/var/log/suricata:z
            - ./suricata/rules:/etc/suricata/rules:z
        command: -i eth0 -s /etc/suricata/rules/local.rules
        restart: unless-stopped # So that the container always boots back up unless explicitly told to shut down

    fluent-bit:
        image: cr.fluentbit.io/fluent/fluent-bit:latest
        container_name: fluent_bit
        volumes:
            - ./suricata/logs:/var/log/suricata:ro
            - ./fluent-bit/fluent-bit.conf:/fluent-bit/etc/fluent-bit.conf:ro
        depends_on:
            - suricata # So that Suricata boots up first
        restart: unless-stopped

    soar-runner:
        build: ./soar
        container_name: soar
        ports:
            - "5000:5000"
        environment:
            - SPLUNK_HEC_URL=https://10.0.0.103:8088/services/collector/raw
            # Points to my Splunk machine on Proxmox
            - SPLUNK_HEC_TOKEN=<token>
        volumes:
            - ./soar:/app
        restart: unless-stopped`

Suricata was picked as the Intrusion Detection System.
Fluent-bit was picked as the log forwarder. (Lightweight and has extensive documentation.)
Soar automates threat response.

Problems that I came across during Phase 2:
Had to explicitly pass the signature path flag ```-s /etc/suricata/rules/local.rules``` so that Suricata could load local.rules
Had to append ```:z``` to the mounted volume paths so that Suricata could read the directories.
I initially had mapped the network interface to be ens18, but it was instead eth0, so I change it to so.


Phase 3: Suricata Rules - Holds custom detection rules
```
# 1. ICMP Ping Detection (The "Hello World" rule to test connectivity)
alert icmp any any -> any any (msg:"ICMP Ping Detected"; sid:1000001; rev:1;)

# 2. SSH Connection Attempt (Flags the initial TCP SYN packet targeting port 22)
alert tcp any any -> any 22 (msg:"SSH Connection Attempt Detected"; flags:S; sid:1000002; rev:1;)

# 3. Nmap SYN Scan Detection (Flags specific TCP window sizes common in stealth scans)
alert tcp any any -> any any (msg:"Suspicious Nmap SYN Scan Detected"; flags:S; window:1024; sid:1000003; rev:1;)
```

#### Rule Anatomy Breakdown:
* **Action & Header:** `alert [protocol] [src_ip] [src_port] -> [dst_ip] [dst_port]` defines traffic directionality and capture action.
* **Rule 1000001 (ICMP):** Baseline connectivity test matching ICMP protocol.
* **Rule 1000002 (SSH Attempt):** Inspects TCP header flags using `flags:S;` to match on SYN packets, isolating connection initialization from established SSH traffic.
* **Rule 1000003 (Nmap Stealth):** Identifies default Nmap SYN scan behavior by inspecting the TCP Window field (`window:1024;`), a signature fingerprint of Nmap's raw packet engine.
* **Rule Metadata:** Unique `sid` (Signature ID >= 1000000 for custom rules) and `rev` (revision tracking).

To verify that these rules are functional, I used my Fedora jumpbox machine to test out ping, ssh, and nmap.
```
tail -n 10 ./suricata/logs/fast.log
10/03/2026-21:59:46.336031  [**] [1:1000001:1] ICMP Ping Detected [**] [Classification: (null)] [Priority: 3] {IPv6-ICMP} fe80:0000:0000:0000:9c09:7bff:fe91:a49d:133 -> ff02:0000:0000:0000:0000:0000:0000:0002:0
10/03/2026-22:07:17.215472  [**] [1:1000001:1] ICMP Ping Detected [**] [Classification: (null)] [Priority: 3] {IPv6-ICMP} fe80:0000:0000:0000:b5ad:81fb:62e8:0826:133 -> ff02:0000:0000:0000:0000:0000:0000:0002:0
10/03/2026-22:24:05.538804  [**] [1:1000001:1] ICMP Ping Detected [**] [Classification: (null)] [Priority: 3] {IPv6-ICMP} fe80:0000:0000:0000:bb50:f46b:cbd9:7044:133 -> ff02:0000:0000:0000:0000:0000:0000:0002:0
10/03/2026-22:28:40.722402  [**] [1:1000001:1] ICMP Ping Detected [**] [Classification: (null)] [Priority: 3] {IPv6-ICMP} fe80:0000:0000:0000:be24:11ff:fe2f:3215:133 -> ff02:0000:0000:0000:0000:0000:0000:0002:0
10/03/2026-22:46:10.274539  [**] [1:1000001:1] ICMP Ping Detected [**] [Classification: (null)] [Priority: 3] {ICMP} 10.0.0.104:8 -> 10.0.0.10:0
10/03/2026-22:46:10.274719  [**] [1:1000001:1] ICMP Ping Detected [**] [Classification: (null)] [Priority: 3] {ICMP} 10.0.0.10:0 -> 10.0.0.104:0
10/03/2026-22:46:21.496163  [**] [1:1000002:1] SSH Connection Attempt Detected [**] [Classification: (null)] [Priority: 3] {TCP} 10.0.0.104:60004 -> 10.0.0.10:22
10/03/2026-22:47:13.962603  [**] [1:1000003:1] Suspicious Nmap SYN Scan Detected [**] [Classification: (null)] [Priority: 3] {TCP} 10.0.0.104:49438 -> 10.0.0.10:80
10/03/2026-22:47:13.962604  [**] [1:1000002:1] SSH Connection Attempt Detected [**] [Classification: (null)] [Priority: 3] {TCP} 10.0.0.104:49438 -> 10.0.0.10:22
10/03/2026-22:47:13.962604  [**] [1:1000003:1] Suspicious Nmap SYN Scan Detected [**] [Classification: (null)] [Priority: 3] {TCP} 10.0.0.104:49438 -> 10.0.0.10:22
```

Phase 4: Suricata Logs - Storage for eve.json telemetry



Phase 5: Soar - Automated response to alerts
