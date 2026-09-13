# Centralized Watchtower

A centralized container automated update service running on the Raspberry Pi infrastructure.

## 1. Architecture Overview

Centralized Watchtower consolidates container update management across the host into a single lightweight daemon.

- **Selective Monitoring**: Uses `WATCHTOWER_LABEL_ENABLE=true` to only monitor and update containers explicitly labeled with `com.centurylinklabs.watchtower.enable=true`.
- **Automatic Pruning**: Removes stale/dangling images with `WATCHTOWER_CLEANUP=true`.
- **Resource Constraints**: Capped at 128MB RAM and 0.5 CPU cores to prevent resource starvation.
- **Service Discovery**: Automatically registers into Consul via Registrator metadata (`SERVICE_NAME=watchtower`, `SERVICE_TAGS=infrastructure,automation`).

---

## 2. Monitored Services

Services enabled for automated updates via label `com.centurylinklabs.watchtower.enable=true`:
- `registry`
- `consul`
- `jenkins`
- `portainer`
- `syslog`
- `Ofelia` (`app`)

Unlabeled or opt-out services (`weather`, `speedtest-arm`, `watchtower` itself) are ignored by Watchtower.

---

## 3. Verification

Check container status and logs on Raspberry Pi:
```bash
docker ps --filter "name=watchtower"
docker logs --tail 50 watchtower
```
