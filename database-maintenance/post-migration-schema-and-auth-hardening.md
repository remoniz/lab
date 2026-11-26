
# Post-Migration MySQL Schema Remediation, Kernel Patching & Storefront Auth Hardening

**File:** `database-maintenance/post-migration-schema-and-auth-hardening.md`  
**Related Runbook:** `database-maintenance/update-woocommerce-sku-duplication.md`  
**Environment:** Self-Hosted WordPress / WooCommerce (`candmmerchandise.com`) & Local AI Node  
**Hypervisor & OS:** Proxmox VE (Dell PowerEdge R720) | Ubuntu 26.04.1 LTS (Ubuntu Pro / ESM)  
**Catalog Scale:** 26,570+ Physical Comic Book SKU Inventory  
**Maintainer:** Robert E. Moniz (CompTIA SecurityX, Security+, Network+, A+)  

---

## 1. Operational Timeline & Incident Background

* **May – June 2026 (Initial Infrastructure Migration):** Migrated the production WordPress and WooCommerce storefront off the legacy Synology NAS onto a dedicated Ubuntu virtual machine running on Proxmox VE, alongside deploying a separate Ubuntu VM for the local AI stack.
* **November 26, 2026 (18:00 – 21:30 UTC / EST Remediation Window):**
  * **18:15 – 19:00:** Configured WooCommerce customer registration, U.S. tax-audit data retention settings, and Cloudflare Turnstile anti-bot protection across storefront authentication endpoints.
  * **19:00 – 19:45:** Applied pending Ubuntu Pro / ESM package updates and executed kernel reboots across both the Web Store VM and the Local AI Server VM (`7.0.0-42-generic` → `7.0.0-44-generic`).
  * **19:45 – 20:30:** Normalized `wp-config.php` SSL/HTTPS reverse-proxy definitions, audited `.htaccess` rewrite rules, and isolated frontend authentication behavior across browsers.
  * **20:30 – 21:30:** Identified missing `AUTO_INCREMENT` and `PRIMARY KEY` table attributes introduced during the May–June 2026 NAS-to-Proxmox SQL import, repaired affected WordPress and WooCommerce HPOS tables, verified end-to-end customer onboarding using a dummy account, and purged temporary SQL snapshots.

---

## 2. Kernel & OS Security Patching (Web Store & Local AI VMs)

Both Ubuntu 26.04.1 LTS Pro virtual machines were patched prior to rebooting so the updated kernel (`7.0.0-44-generic`) loaded cleanly on the first restart:

* **Web Store VM:** Applied 24 pending updates (including 8 standard security updates) and rebooted from `7.0.0-42-generic` into `7.0.0-44-generic`, remediating **518 CVEs** (4 Critical, 102 High, 175 Medium, 233 Low, 4 Negligible).
* **Local AI Server VM:** Applied 15 pending updates and rebooted from `7.0.0-42-generic` into `7.0.0-44-generic` to maintain identical patch and kernel baselines across lab nodes.

```bash
# 1. Apply all pending repository and ESM security updates first
sudo apt update && sudo apt upgrade -y

# 2. Reboot into the updated kernel
sudo reboot

# 3. Verify running kernel and zero remaining updates
uname -r
# Output: 7.0.0-44-generic
