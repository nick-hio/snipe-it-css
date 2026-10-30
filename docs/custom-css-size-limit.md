# Raise the Custom CSS size limit

Snipe-IT keeps **Custom CSS** in the `custom_css` column of its `settings` table. That column is `TEXT`, which holds **65,535 bytes**. If you save more, Snipe-IT reports success but cuts the CSS at 65,535 bytes. Change the column to `MEDIUMTEXT` to hold up to **16,777,215 bytes** (16 MiB).

The change is in the database only. You keep the official Docker image, and you do not change Snipe-IT's code. You paste and save CSS on the Branding page as before.

Tested end to end with the official `snipe/snipe-it` image, **v8.8.3 (build 26143)**, on **MariaDB 11.4** and **MySQL 8.4**. The SQL also works on MariaDB 10.11 and MySQL 8.0. After the change, all table data was unchanged, and stylesheets of 240–300 KB saved and showed in full. The change stayed after the Snipe-IT container was recreated.

## Before you start

- You need a Snipe-IT administrator account and shell access to the Docker host.
- You need a database account that can change Snipe-IT's tables. Use `root` (`MYSQL_ROOT_PASSWORD`) or the Snipe-IT database user (`DB_USERNAME` and `DB_PASSWORD`).
- Snipe-IT is unavailable for a few minutes. The database stays online.

These steps use Docker commands. For a database outside Docker, run the same SQL in the `mariadb` or `mysql` command-line client. In a graphical client, end the step 5 query with `;` instead of `\G`.

## Change the column

1. **Back up Snipe-IT.**

   Open **Settings → Backups**. Generate a backup, then download it. Do not continue without a backup.

2. **Find the container names.**

   ```bash
   docker ps --format 'table {{.Names}}\t{{.Image}}'
   ```

   Set `APP` to the Snipe-IT container (`snipe/snipe-it` image). Set `DB` to the database container (`mariadb` or `mysql` image). Docker Compose names usually end in `-app-1` and `-db-1`.

   ```bash
   APP=snipe-it-app-1
   DB=snipe-it-db-1
   ```

   Keep this terminal open. The later steps use these variables.

3. **Stop Snipe-IT.**

   ```bash
   docker stop "$APP"
   ```

   This prevents writes during the change. It also works for Docker Compose containers.

4. **Open a database shell.**

   ```bash
   docker exec -it "$DB" mariadb --user=root --password
   ```

   Type the password at the prompt. The input is hidden. To use the Snipe-IT database user, replace `root` with that user name. If you see `executable file not found`, use `mysql` instead of `mariadb`.

   If you cannot sign in, run `docker start "$APP"` to restore Snipe-IT before you troubleshoot.

5. **Inspect the column.**

   Paste this at the `MariaDB [(none)]>` or `mysql>` prompt:

   ```sql
   SELECT
     TABLE_SCHEMA, TABLE_NAME, COLUMN_TYPE, IS_NULLABLE,
     COLUMN_DEFAULT, COLLATION_NAME, COLUMN_COMMENT,
     CONCAT('USE `', REPLACE(TABLE_SCHEMA, '`', '``'), '`;') AS use_database,
     CONCAT('ALTER TABLE `', REPLACE(TABLE_NAME, '`', '``'),
            '` MODIFY COLUMN `custom_css` MEDIUMTEXT CHARACTER SET ',
            CHARACTER_SET_NAME, ' COLLATE ', COLLATION_NAME, ' NULL;') AS alteration
   FROM information_schema.COLUMNS
   WHERE COLUMN_NAME = 'custom_css'
     AND TABLE_NAME LIKE '%settings'\G
   ```

   Make sure that the result shows:

   | Field | Expected value |
   | --- | --- |
   | `COLUMN_TYPE` | `text` |
   | `IS_NULLABLE` | `YES` |
   | `COLUMN_DEFAULT` | `NULL` |
   | `COLUMN_COMMENT` | Empty |

   - **More than one row:** use the row where `TABLE_SCHEMA` is Snipe-IT's `DB_DATABASE` value and `TABLE_NAME` is its `DB_PREFIX` value followed by `settings`. With no prefix, `TABLE_NAME` is `settings`. If no row matches, type `QUIT;`, then run `docker start "$APP"`.
   - **No rows:** you are signed in to the wrong database server, or the user cannot see Snipe-IT's tables. Type `QUIT;`, then run `docker start "$APP"`.
   - **`COLUMN_TYPE` is already `mediumtext` or `longtext`:** no change is needed. Type `QUIT;`, then go to step 8.
   - **Other values are different:** stop. The statement in step 6 would change them. Type `QUIT;`, then run `docker start "$APP"`.

   The generated statements keep your database name, table prefix, character set and collation.

6. **Change the column.**

   Copy the `use_database` value from step 5 and run it. Then copy the `alteration` value and run it. They look like this:

   ```sql
   USE `snipeit`;
   ALTER TABLE `settings` MODIFY COLUMN `custom_css` MEDIUMTEXT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL;
   ```

   Use your generated values, not this example. The change is usually quick, but it can wait for database locks. If it fails, type `QUIT;`, then run `docker start "$APP"` before you troubleshoot.

7. **Make sure that the change is correct.**

   ```sql
   SELECT TABLE_NAME, COLUMN_TYPE, IS_NULLABLE, CHARACTER_OCTET_LENGTH
   FROM information_schema.COLUMNS
   WHERE TABLE_SCHEMA = DATABASE()
     AND COLUMN_NAME = 'custom_css';
   ```

   Expected values: `COLUMN_TYPE` is `mediumtext`, `IS_NULLABLE` is `YES` and `CHARACTER_OCTET_LENGTH` is `16777215`. If the result is empty, run the `use_database` statement from step 5 again.

   Close the database shell:

   ```sql
   QUIT;
   ```

8. **Start Snipe-IT.**

   ```bash
   docker start "$APP"
   ```

   Startup can take a minute. If the page does not load, look at the logs:

   ```bash
   docker logs --tail 50 "$APP"
   ```

9. **Save your CSS.**

   1. Sign in. Open some pages, such as **Assets** and **Users**, to make sure that they work.
   2. Open **Settings → Branding**. Paste your CSS in **Custom CSS**, then save.
   3. Reload the page. Make sure that the end of your CSS is still in the field.
   4. Open a page that uses a rule from the end of your CSS. Make sure that the rule applies.

To see the stored size, open the database shell again, run the `use_database` statement, and run this query. Replace `settings` with your `TABLE_NAME` if it is different.

```sql
SELECT OCTET_LENGTH(custom_css) AS css_bytes FROM settings;
```

The stored size can be larger than your file. Browsers send each line break as two bytes (CRLF).

## Upgrades and restores

- **Upgrades:** Snipe-IT runs database migrations when its container starts. In v8.8.3, only the migration that added the column in 2015 changes `custom_css`, so upgrades keep `MEDIUMTEXT`. A future release can change this. After each upgrade, open a database shell (step 4) and run the step 5 query. `COLUMN_TYPE` must still be `mediumtext`.
- **Restores:** A backup made after the change keeps `MEDIUMTEXT`. A backup made before the change restores `TEXT`. Make the change again after you restore an old backup.
- **New installations:** Snipe-IT creates the column as `TEXT`. Do all the steps again.

## Undo the change

1. Reduce the saved CSS to 65,535 bytes or less. Use the stored size query above to measure it.
2. Do steps 1 to 4 again.
3. Run the step 5 query. `COLUMN_TYPE` now shows `mediumtext`. Ignore the "already `mediumtext`" rule.
4. Run the `use_database` value. Then run the `alteration` value, with `MEDIUMTEXT` replaced by `TEXT`.
5. Do step 7, but expect `text` and `65535`. Then do step 8.

If the CSS is too large, MariaDB and MySQL usually stop with a `Data too long` error and do not change the column. If strict SQL mode is off, they can cut the CSS without an error. Always reduce the CSS first.

## Limits that still apply

- **Request size:** PHP accepts 8 MB per request by default in the Docker image. Set `PHP_UPLOAD_LIMIT` (in MB) to raise it. If a save is too large, Snipe-IT shows "Your form session has expired" and saves nothing. A reverse proxy (Nginx allows 1 MB by default) and the database (`max_allowed_packet`) can also reject a large save.
- **PHP memory:** near 16 MB of CSS, the Branding page fails to load (HTTP 500) with the image's 128 MB `memory_limit`. Keep the CSS well below that.
- **Page size:** Snipe-IT puts all of the Custom CSS into every page. A larger stylesheet makes every page larger.
- **CSS filtering:** Snipe-IT still filters Custom CSS when it shows a page. For example, it removes `@import` rules. It shows no Custom CSS at all if the CSS contains a backslash (`\`).
