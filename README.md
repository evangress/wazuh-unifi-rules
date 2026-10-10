# wazuh-unifi-rules
An attempt to create a complete rule set for CEF formatted syslogs originating from Ubiquiti Unifi network equipment

Requires the `ubiquiti` decoder from [wazuh-unifi-decoder](https://github.com/evangress/wazuh-unifi-decoder).

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

When adding a rule, add a sample for it in the decoder repo.
