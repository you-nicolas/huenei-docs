# Incident Postmortem: Keycloak Production Outage & Data Loss

**Date:** May 14, 2026  
**Status:** Resolved (Restored from April 14th Backup)  
**Service:** Keycloak Identity Provider (`keycloak-prod-huenei.net`)  
**Impact:** 4 days of downtime + ~30 days of data loss (April 15 – May 11).

---

## 1. Summary
On May 11, the Keycloak production database experienced a filesystem-level crash. During troubleshooting on May 14, a `docker volume prune -af` command was executed while the services were in a stopped state. Due to the containers being stopped, Docker identified the volumes as "unreferenced" and deleted them. The service was restored using a backup from a recovery server, resulting in the recovery of all realms but with a one-month gap in recent configuration changes and user registrations.

## 2. Timeline (All times EDT)
*   **May 11, 21:34:** Postgres database crashed. Logs indicate: `performing immediate shutdown because data directory lock file is invalid`.
*   **May 14, 09:00:** Troubleshooting initiated. Service confirmed down with 502 Bad Gateway.
*   **May 14, 10:45:** Cleanup commands (`docker system prune`, `docker volume prune`) were executed to free up disk space and reset the environment.
*   **May 14, 11:02:** `docker compose up -d` executed. Postgres initialized a **new, empty database** because the original volume directory had been removed by the prune command.
*   **May 14, 11:20:** Root cause identified: Data loss due to volume pruning of stopped containers.
*   **May 14, 14:00:** Recovery server (192.169.0.103) provisioned with April 14th backup.
*   **May 14, 15:30:** Data successfully synced via `rsync`, permissions corrected (UID 999), and Keycloak service restored.

## 3. Root Cause Analysis (RCA)
### Primary Trigger
The initial crash was caused by a filesystem/lock error in Postgres. This is often related to an abrupt system shutdown, OOM (Out of Memory) killer, or disk I/O pressure.

### Data Loss Mechanism
The data loss occurred due to a combination of two factors:
1.  **Stopped Container State:** `docker volume prune` only protects volumes attached to *running* or *paused* containers. Since the Keycloak containers were `Exited`, Docker flagged their volumes as "unused."
2.  **Internal Path Dependency:** The `docker-compose.yml` was configured to bind-mount a path inside Docker's internal storage: `/var/lib/docker/volumes/keycloack_postgres_data/_data`. 
    *   *Note:* A typo in the path (`keycloack` with a **c**) contributed to confusion during manual path validation.

## 4. Resolution & Recovery
*   **Recovery:** A manual `rsync` was performed from the recovery server (103) to the production server.
*   **Permissions:** Data ownership was restored to UID `999` (Postgres) and GID `999`. 
*   **Validation:** Verified that the "Master" and production-specific realms are visible and accessible.

## 5. Lessons Learned
1.  **Prune Caution:** `docker volume prune` is highly dangerous in production, especially when troubleshooting "Down" services. It should be avoided unless every container is confirmed as "Running."
2.  **Architectural Debt:** Storing production data inside `/var/lib/docker/volumes/` via bind-mounts creates a "hidden" dependency. If Docker manages the parent folder, manual file operations can conflict with Docker's internal logic.
3.  **Backup Frequency:** The latest available backup was 30 days old. This gap is the primary reason for the data loss window.

## 6. Action Items (Prevention Plan)
| Action Item | Responsibility | Priority |
| :--- | :--- | :--- |
| **Move Data out of /var/lib/docker:** Relocate production DB volumes to `/srv/keycloak/data` or `/opt/data`. This isolates data from `docker prune` commands. | DevSecOps | **High** |
| **Fix Path Typos:** Correct `keycloack` to `keycloak` in all `.yml` and scripts to prevent "missing directory" false positives. | DevSecOps | Medium |
| **Automate Backups:** Implement a daily `pg_dump` cron job that exports to an external S3/off-site bucket. | Infra Team | **High** |
| **Implement Monitoring:** Setup alerts for "Container Down" or "Postgres Fatal" logs to catch crashes closer to the T=0 event. | Monitoring | Medium |
| **Update Documentation:** Add a "Production Troubleshooting Safety" section to the internal Wiki, specifically banning `prune` commands on live nodes. | Team Lead | Medium |

---
**Conclusion:** The service is back online. While the loss of 30 days of data is regrettable, the recovery of the core realms ensures that the majority of identity services are functional without a total rebuild.
