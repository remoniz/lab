# WooCommerce Duplicate SKU Audit & Cleanup Runbook

## Overview
SQL workflow for auditing and safely soft-deleting duplicate WooCommerce products that share an identical `_sku` value while preserving the most recently updated record (`MAX(ID)`).

## Safety Protocol
1. **Back up the database** prior to executing any `UPDATE` statements.
2. **Audit first:** Run the read-only `SELECT` verification queries to inspect paired records and ensure empty/whitespace SKUs are excluded.
3. **Soft-delete to Trash:** Move older duplicates (`post_status = 'trash'`) rather than running a hard `DELETE`, allowing review and WP-CLI cleanup afterward.

---

## Step 1: Audit for Blank or Whitespace-Only SKUs
Ensure records with missing SKUs are not grouped together as false duplicates:

```sql
SELECT COUNT(*) AS blank_sku_count
FROM wp_posts p
JOIN wp_postmeta pm ON p.ID = pm.post_id AND pm.meta_key = '_sku'
WHERE p.post_type = 'product'
  AND p.post_status IN ('publish', 'draft')
  AND TRIM(pm.meta_value) = '';
```

## Step 2: Side-by-Side Duplicate Verification
Lists every duplicate SKU pair, marking the newest record (`MAX(ID)`) to keep and older copies to trash:

```sql
SELECT 
    pm.meta_value AS sku,
    p.ID AS product_id,
    p.post_title,
    p.post_status,
    p.post_date,
    CASE 
        WHEN p.ID = newest_copy.max_id THEN 'KEEP (Newest)'
        ELSE 'DUPLICATE (Will Trash)'
    END AS action_status
FROM wp_posts p
JOIN wp_postmeta pm ON p.ID = pm.post_id AND pm.meta_key = '_sku'
JOIN (
    SELECT pm2.meta_value AS dup_sku, MAX(p2.ID) AS max_id
    FROM wp_posts p2
    JOIN wp_postmeta pm2 ON p2.ID = pm2.post_id AND pm2.meta_key = '_sku'
    WHERE p2.post_type = 'product'
      AND p2.post_status IN ('publish', 'draft')
      AND TRIM(pm2.meta_value) != ''
    GROUP BY pm2.meta_value
    HAVING COUNT(*) > 1
) AS newest_copy ON pm.meta_value = newest_copy.dup_sku
WHERE p.post_type = 'product'
  AND p.post_status IN ('publish', 'draft')
ORDER BY pm.meta_value ASC, p.ID DESC;
```

## Step 3: Move Older Duplicates to Trash
Soft-deletes older duplicate product IDs while retaining the newest record per SKU:

```sql
UPDATE wp_posts
SET post_status = 'trash'
WHERE ID IN (
  SELECT * FROM (
    SELECT p.ID
    FROM wp_posts p
    JOIN wp_postmeta pm ON p.ID = pm.post_id AND pm.meta_key = '_sku'
    WHERE p.post_type = 'product'
      AND p.post_status IN ('publish', 'draft')
      AND TRIM(pm.meta_value) != ''
      AND p.ID NOT IN (
        SELECT * FROM (
          SELECT MAX(p2.ID)
          FROM wp_posts p2
          JOIN wp_postmeta pm2 ON p2.ID = pm2.post_id AND pm2.meta_key = '_sku'
          WHERE p2.post_type = 'product'
            AND p2.post_status IN ('publish', 'draft')
            AND TRIM(pm2.meta_value) != ''
          GROUP BY pm2.meta_value
        ) AS keep_ids
      )
  ) AS dup_ids
);
```

## Step 4: Purge Trashed Records via WP-CLI (SSH Terminal)
> **Note:** Run this command in your server's **SSH terminal** inside `/var/www/html`—**not** in phpMyAdmin.

After verifying the catalog, permanently purge trashed products and associated `wp_postmeta` rows via WP-CLI to avoid web/database lock timeouts:

```bash
cd /var/www/html
wp post delete $(wp post list --post_type='product' --post_status='trash' --format=ids) --force
```

### Alternative: Purge Trashed Records via phpMyAdmin (SQL)
If SSH/WP-CLI is unavailable, run these SQL statements in **phpMyAdmin** to purge trashed products and their orphaned metadata:

```sql
DELETE pm FROM wp_postmeta pm
JOIN wp_posts p ON pm.post_id = p.ID
WHERE p.post_type = 'product' AND p.post_status = 'trash';

DELETE tr FROM wp_term_relationships tr
JOIN wp_posts p ON tr.object_id = p.ID
WHERE p.post_type = 'product' AND p.post_status = 'trash';

DELETE FROM wp_posts
WHERE post_type = 'product' AND post_status = 'trash';
```

```
