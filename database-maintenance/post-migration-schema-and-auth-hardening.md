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
```

---

## 3. Root Cause Analysis: NAS-to-Proxmox Database Schema Corruption

### Observed Symptoms
1. **Frontend Login Loops (`Last Login: Never`):** Newly created accounts failed to authenticate on `/my-account/`, silently reloading the login screen with `Last Login: Never` in **Users > All Users** (via the *When Last Login* plugin), whereas pre-migration administrator accounts authenticated normally.
2. **User Role Count Discrepancy:** WordPress **Users > All Users** reported `All (3)` despite 4 user rows existing in `wp_users`.

### Root Cause
During the May–June 2026 SQL migration from the Synology NAS to the Proxmox VM, `AUTO_INCREMENT` (and `PRIMARY KEY`) constraints were stripped from several core WordPress and WooCommerce tables. When new users were inserted into `wp_users`, subsequent inserts into `wp_usermeta` failed or wrote `umeta_id = 0`, leaving `wp_capabilities` as `NULL`. Without a valid serialized role array (`a:1:{s:8:"customer";b:1;}`), WordPress omitted those users from role counts and rejected session creation.

A full schema audit against `information_schema.COLUMNS` confirmed that `wp_wc_orders` (WooCommerce High-Performance Order Storage), `wp_comments`, `wp_commentmeta`, `wp_wc_download_log`, and `wp_wc_admin_note_actions` also lost `AUTO_INCREMENT` during the same migration.

---

## 4. Database Schema Remediation Procedure

### Step 1: Extract `DB_NAME` Environment Variable & Create Pre-Change Backup
```bash
DB_NAME=$(grep DB_NAME /var/www/html/wp-config.php | cut -d \' -f 4)
sudo mysqldump "$DB_NAME" > ~/wp_backup_schema_fix.sql
```

### Step 2: Inspect User Capabilities & Restore `AUTO_INCREMENT` on `wp_usermeta` / `wp_users`
```bash
# Audit users missing wp_capabilities
sudo mysql -e "USE $DB_NAME; \
SELECT u.ID, u.user_login, u.user_email, m.meta_value AS capabilities \
FROM wp_users u \
LEFT JOIN wp_usermeta m ON u.ID = m.user_id AND m.meta_key = 'wp_capabilities';"

# Clean zero-ID rows and restore PRIMARY KEY + AUTO_INCREMENT
sudo mysql -e "USE $DB_NAME; \
DELETE FROM wp_usermeta WHERE umeta_id = 0; \
ALTER TABLE wp_usermeta MODIFY umeta_id bigint(20) unsigned NOT NULL AUTO_INCREMENT PRIMARY KEY; \
ALTER TABLE wp_users MODIFY ID bigint(20) unsigned NOT NULL AUTO_INCREMENT;"
```

### Step 3: Backfill Missing `customer` Capabilities for Test Accounts
```bash
sudo mysql -e "USE $DB_NAME; \
INSERT INTO wp_usermeta (user_id, meta_key, meta_value) \
SELECT ID, 'wp_capabilities', 'a:1:{s:8:\"customer\";b:1;}' \
FROM wp_users \
WHERE ID NOT IN (SELECT user_id FROM wp_usermeta WHERE meta_key = 'wp_capabilities');"
```

### Step 4: Audit & Repair Remaining WooCommerce HPOS & Core Tables
```bash
# Query all primary-key columns missing AUTO_INCREMENT
sudo mysql -e "SELECT TABLE_NAME, COLUMN_NAME, EXTRA \
FROM information_schema.COLUMNS \
WHERE TABLE_SCHEMA = '$DB_NAME' \
  AND COLUMN_KEY = 'PRI' \
  AND EXTRA NOT LIKE '%auto_increment%';"

# Restore AUTO_INCREMENT on single-key WordPress and WooCommerce tables
sudo mysql -e "USE $DB_NAME; \
DELETE FROM wp_wc_orders WHERE id = 0; \
ALTER TABLE wp_wc_orders MODIFY id bigint(20) unsigned NOT NULL AUTO_INCREMENT; \
DELETE FROM wp_comments WHERE comment_ID = 0; \
ALTER TABLE wp_comments MODIFY comment_ID bigint(20) unsigned NOT NULL AUTO_INCREMENT; \
DELETE FROM wp_commentmeta WHERE meta_id = 0; \
ALTER TABLE wp_commentmeta MODIFY meta_id bigint(20) unsigned NOT NULL AUTO_INCREMENT; \
DELETE FROM wp_wc_download_log WHERE download_log_id = 0; \
ALTER TABLE wp_wc_download_log MODIFY download_log_id bigint(20) unsigned NOT NULL AUTO_INCREMENT; \
DELETE FROM wp_wc_admin_note_actions WHERE action_id = 0; \
ALTER TABLE wp_wc_admin_note_actions MODIFY action_id bigint(20) unsigned NOT NULL AUTO_INCREMENT;"
```
*(Note: Composite-key lookup tables such as `wp_wc_order_product_lookup`, `wp_wc_product_attributes_lookup`, and WP All Import hash tables intentionally omit `AUTO_INCREMENT` and must be left unchanged.)*

### Step 5: Remove Temporary SQL Dump
```bash
rm -f ~/wp_backup_schema_fix.sql
```

---

## 5. SSL/HTTPS Reverse-Proxy & `.htaccess` Verification

### `wp-config.php` HTTPS Definitions
Updated hardcoded `http://` constants to `https://` and ensured reverse-proxy `X-Forwarded-Proto` headers populate `$_SERVER['HTTPS'] = 'on'` so WordPress issues secure `wordpress_logged_in_*` cookies consistently:

```php
if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && strpos($_SERVER['HTTP_X_FORWARDED_PROTO'], 'https') !== false) {
    $_SERVER['HTTPS'] = 'on';
}

define('WP_HOME', '[https://candmmerchandise.com](https://candmmerchandise.com)');
define('WP_SITEURL', '[https://candmmerchandise.com](https://candmmerchandise.com)');
define('FORCE_SSL_ADMIN', true);
```

### `.htaccess` Audit
Verified `/var/www/html/.htaccess` contains no conflicting `301` redirects that strip `POST` payloads on `/my-account/`, retaining only the standard REST API authorization header rule (`RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]`).

---

## 6. Storefront Privacy, Audit Compliance & Anti-Bot Hardening

* **WooCommerce Accounts & Privacy (`WooCommerce > Settings > Accounts & Privacy`):**
  * Enabled guest checkout alongside customer registration during checkout and on `/my-account/`, using automatic username generation and password-setup email links.
  * Disabled automated personal data erasure on request and left personal data retention timers blank to preserve immutable transaction records required for U.S. sales tax compliance and financial audits.
* **Cloudflare Turnstile (`Settings > Cloudflare Turnstile`):**
  * Enabled Turnstile challenge verification across **WordPress Login**, **WooCommerce Login**, **WooCommerce Registration**, **WooCommerce Reset Password**, and **WooCommerce Checkout**.
  * Disabled **Defer Scripts** under Turnstile performance settings so challenge tokens populate reliably prior to form `POST` submission even under strict browser tracking protection.
* **SKU Deduplication Reference:** Incremental inventory imports (`AA`, `AB`, `AC`...) continue to match **Existing Items** by Custom Field `_sku` per `database-maintenance/update-woocommerce-sku-duplication.md`.

---

## 7. Dummy Account Verification & Post-Deployment Sign-Off

To validate end-to-end account creation, role assignment, and session persistence after the database schema repair:

1. **Dummy Account Creation:** Created a dedicated non-admin test customer account (`remoniztest`) alongside test user `rob` and verified `Users > All Users` updated to `All (4)` with valid `Customer` roles.
2. **Frontend Authentication & Persistence Test:** Authenticated via `/my-account/` in Firefox, updated **Account details** (Display Name) and **Billing address** fields, and confirmed green confirmation banners, persistent values across dashboard tabs, and an updated timestamp in **Last Login**.
3. **Final Hygiene:** Confirmed `~/wp_backup_schema_fix.sql` was deleted from the VM home directory.
