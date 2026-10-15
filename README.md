# Wazuh rules for Ubiquiti UniFi (CEF syslog / SIEM integration)

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Wazuh 4.14](https://img.shields.io/badge/Wazuh-4.14-3595F9)](https://wazuh.com/)
[![UniFi Network 10.x](https://img.shields.io/badge/UniFi%20Network-10.x-0559C9)](https://help.ui.com/hc/en-us/articles/33349041044119-UniFi-System-Logs-SIEM-Integration)

Alert rules that turn **Ubiquiti UniFi** CEF syslog into **Wazuh** alerts. Send your UniFi Network logs to Wazuh and get alerts for IDS/IPS threats, honeypot hits, admin logins, configuration and firewall changes, WAN outages, and access points or switches going offline.

Works with any device that runs UniFi Network and exports logs through **Integration → System Logging / SIEM**: UniFi Dream Machine (UDM, UDM Pro, UDM SE), Cloud Gateway (UCG Ultra, UCG Max), Cloud Key, UniFi OS Server, and self-hosted controllers.

These rules need the `ubiquiti` decoder from the companion repo **[wazuh-unifi-decoder](https://github.com/evangress/wazuh-unifi-decoder)**. Install both.

## What it alerts on

| Area | Events | Wazuh level |
|---|---|---|
| Intrusion prevention (IDS/IPS) | Threat Detected and Blocked, high-risk intrusions | 12–13 |
| Honeypot and rogue AP | Honeypot triggered, WiFi impersonation (rogue SSID) | 10 |
| Firewall | Firewall events; firewall policy created, changed or removed | 5–7 |
| Admin audit | Admin logins, settings modified, created or removed | 5 |
| Internet / WAN | Internet down, repeated disconnections, failover, high latency, packet loss, restored | 5–12 |
| Devices | Device offline, multiple devices offline, adopted, unadopted device on network, mesh changes | 3–12 |
| Updates | Firmware and UniFi application updates | 5 |
| Anything else | Mapped from UniFi's own CEF severity: 9–10 → 12, 7–8 → 10, 4–6 → 5 | 5–12 |

Routine client connect, disconnect and roam events are stored at level 2, so they are searchable but don't alert. Alerts carry the `unifi` group, and security, authentication and config-change rules add the standard Wazuh groups (`ids`, `attack`, `firewall`, `authentication_success`, `config_changed`) so they show up in the built-in dashboards.

The rules use IDs **100010–100399**. The full list with descriptions is in [`ubiquiti_rules.xml`](ubiquiti_rules.xml).

## Installation

Tested on Wazuh 4.14.8 with UniFi Network 10.x. Run these steps on the Wazuh **manager**, as root.

### 1. Install the decoder and rules

```sh
cd /var/ossec/etc
curl -fsSL -o decoders/ubiquiti_decoders.xml \
  https://raw.githubusercontent.com/evangress/wazuh-unifi-decoder/main/ubiquiti_decoders.xml
curl -fsSL -o rules/ubiquiti_rules.xml \
  https://raw.githubusercontent.com/evangress/wazuh-unifi-rules/main/ubiquiti_rules.xml
chown wazuh:wazuh decoders/ubiquiti_decoders.xml rules/ubiquiti_rules.xml
chmod 660 decoders/ubiquiti_decoders.xml rules/ubiquiti_rules.xml
```

The default `ossec.conf` already loads every file in `etc/decoders` and `etc/rules`, so no `<ruleset>` change is needed. You can also upload both files in the Wazuh dashboard under **Server management → Decoders** and **Rules** (custom files).

If your own custom rules already use IDs 100010–100399, renumber one set to avoid duplicate-ID warnings.

### 2. Accept syslog from the UniFi console

Add a syslog listener to `/var/ossec/etc/ossec.conf`, inside `<ossec_config>`. Set `allowed-ips` to the address of the device that runs UniFi Network (Cloud Key, UDM/UCG, or a self-hosted server). Wazuh silently drops syslog from any address that isn't listed.

```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>192.0.2.10</allowed-ips>  <!-- your UniFi console's IP -->
</remote>
```

Open 514/udp in the manager's firewall, for example `ufw allow from 192.0.2.10 to any port 514 proto udp`, or with firewalld: `firewall-cmd --permanent --add-port=514/udp && firewall-cmd --reload`.

### 3. Restart and check for errors

```sh
systemctl restart wazuh-manager
grep -E 'analysisd.*(ERROR|CRITICAL|WARNING)' /var/ossec/logs/ossec.log | tail
```

### 4. Verify with wazuh-logtest

Paste a sample event into `/var/ossec/bin/wazuh-logtest`:

```
Oct 10 16:00:00 unifi CEF:0|Ubiquiti|UniFi Network|10.6.106|512|Device Offline|8|UNIFIcategory=UniFi Devices UNIFIdeviceName=Lobby AP UNIFIdeviceModel=U6-Pro UNIFIdeviceIp=192.0.2.20 msg=Lobby AP went offline.
```

You should see `name: 'ubiquiti'` in Phase 2 and rule `100301`, level `10`, in Phase 3.

### 5. Point UniFi at Wazuh

In UniFi Network, go to **Integration → System Logging / SIEM**, choose **SIEM Server**, enter the Wazuh manager's IP and port **514**, and choose the log categories to export. Events then appear as alerts with the `unifi` group.

Leave the controller's syslog **debug** option off. It makes devices send verbose non-CEF logs that these rules don't use, and those logs include device auth keys.

### Troubleshooting

- **Nothing arrives:** first check that the console's IP is in `allowed-ips` and that the firewall allows 514/udp. Then run `tcpdump -ni any udp port 514` on the manager to see whether packets reach it at all.
- **Events arrive but don't alert:** set `<logall_json>yes</logall_json>` in `<global>` temporarily and look at `/var/ossec/logs/archives/archives.json`. An empty `"decoder"` means the decoder isn't installed, or is an old version (from before October 2026) that doesn't match syslog-framed CEF.

### Upgrading

Repeat step 1 for both files, then restart the manager.

## Testing

The test lab and samples live in the decoder repo:

- [`lab/docker-compose.yml`](https://github.com/evangress/wazuh-unifi-decoder/tree/main/lab): a Wazuh manager that mounts both repos. Clone them side by side, or set `UNIFI_RULES_REPO`.
- [`samples/events.log`](https://github.com/evangress/wazuh-unifi-decoder/blob/main/samples/events.log) and `test.py`: expected rule ID and level per event.

```sh
cd ../wazuh-unifi-decoder
docker compose -f lab/docker-compose.yml up -d
./test.py
docker compose -f lab/docker-compose.yml down -v
```

## Contributing

Event names and fields come from real controller output, which differs from Ubiquiti's documentation. If your controller sends an event these rules miss or misclassify, open an issue with an **anonymized** copy of the log line (replace hostnames, client and device names, MAC addresses, IPs and ISP names). When adding a rule, add a sample for it in the decoder repo.

## Related

- [wazuh-unifi-decoder](https://github.com/evangress/wazuh-unifi-decoder): the decoder these rules depend on
- [UniFi System Logs & SIEM Integration](https://help.ui.com/hc/en-us/articles/33349041044119-UniFi-System-Logs-SIEM-Integration) (Ubiquiti docs)
- [Wazuh custom rules](https://documentation.wazuh.com/current/user-manual/ruleset/rules/custom.html) (Wazuh docs)

## License

[MIT](LICENSE)
