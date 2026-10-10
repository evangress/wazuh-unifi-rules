# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A single Wazuh rules file, `ubiquiti_rules.xml`, that turns UniFi CEF syslog events into alerts. It depends on the `ubiquiti` decoder from the sibling repo [wazuh-unifi-decoder](https://github.com/evangress/wazuh-unifi-decoder). Rules match only on fields that decoder extracts, so the decoder's field names (`event.name`, `event.severity`, `unifi_category`, `unifi_settings_section`, …) are this repo's API. When a rule needs a key that isn't decoded yet, add the decoder there first.

## Testing

All test tooling lives in the decoder repo; clone it next to this one:

```sh
cd ../wazuh-unifi-decoder
docker compose -f lab/docker-compose.yml up -d   # Wazuh manager mounting both repos
./test.py                                       # restarts Wazuh, checks samples/events.log
docker compose -f lab/docker-compose.yml down -v
```

`samples/events.log` in the decoder repo pins the expected rule ID and level for each sample. Every new or changed rule needs a sample there. Anonymize live events before adding them: hostnames, client and device names, MAC addresses (including the short `xx:xx` suffixes UniFi appends to names), private and public IPs, and ISP names. `test.py` also fails on new analysisd warnings, such as duplicate rule IDs.

## Rule structure

- **Base rule 100010** (`<decoded_as>ubiquiti</decoded_as>`, level 2) matches every UniFi event. Every other rule is a child of it via `<if_sid>100010</if_sid>`, except where a rule must lower a broader match (see below). Level 2 means the event is stored but doesn't alert; the WiFi and wired client connect, disconnect and roam events stay there on purpose.
- **ID ranges** (file reserves 100010–100399): 1001xx security, 1002xx admin and config audit, 1003xx WAN, devices and updates, 100390–100392 severity fallbacks.
- **Severity fallbacks** map UniFi's CEF severity to a level for events without a specific rule: 9–10 → 12, 7–8 → 10, 4–6 → 5, and 0–3 stay at the base level 2.
- **Match on exact event names** (`<field name="event.name">^Device Offline$</field>`). Take the names from real controller output, not UniFi's docs, which differ. For example, event 544 is documented as "Admin Accessed UniFi Network" but real controllers send "Network Accessed"; rule 100200 matches both.
- Descriptions interpolate decoded fields, for example `$(unifi_device_name)`. A field missing from an event renders as empty text.

## How Wazuh picks a rule (the non-obvious part)

Child rules of the same parent are tried **from highest level to lowest**, not in file order. On a **tie, the earlier rule in the file wins**. Consequences:

- **A specific rule must have a level at least equal to the severity fallback its event would hit, and must sit above the fallbacks in the file.** UniFi sends a fixed severity per event type, so look at the severity in a real sample. Otherwise the generic fallback silently wins: Device Offline arrives at severity 8 (fallback level 10), so its rule 100301 must be level 10 or higher. Tests with made-up low severities hide this; use real severities in samples.
- **To lower the level of a broader match, make the rule a child of that rule**, not a sibling. Example: firewall rule 100103 (level 5) is a child of generic security rule 100104 (level 7).
- **To raise the level for a subset, use a higher-level sibling.** Example: firewall policy changes, rule 100204 at level 7, override the level-5 config rules 100201–100203.

UniFi IPS events ("Threat Detected and Blocked") carry no `UNIFIsubCategory` or `UNIFIrisk`, so 100105 matches them by name. Rules 100100 and 100101, which require those fields, have no live sample yet.

## Repo notes

`.gitignore` ignores every subdirectory (`*/`); add a `!dir/` exception before creating one. The tracked `.idea/` files predate that rule.
