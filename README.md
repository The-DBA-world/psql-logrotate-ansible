PostgreSQL Logrotate Configuration
Overview
This document details the logrotate configuration for managing PostgreSQL log files, deployed via Ansible playbooks. The configuration is designed to ensure efficient log management, prevent disk space issues, and maintain security and operational continuity for a PostgreSQL database environment.
The setup is applied to log files located in the directory defined by the pg_log_dir variable (e.g., /data/test/postgres/15/pg_dbtest_anil/log/), which is dynamically generated based on the PostgreSQL environment, version, and database name.
File Structure
The configuration is part of an Ansible deployment with the following structure:
project_root/
├── pgsql-06-logrotate-config.yml
├── logrotate_config_deployment/
│   └── logrotate_config.yml


pgsql-06-logrotate-config.yml: Main playbook that deploys the logrotate configuration.
logrotate_config.yml: Contains variables (e.g., pg_log_dir, postgres_user, postgres_group).

Logrotate Configuration
The logrotate configuration is written to /etc/logrotate.d/postgresql and targets all .log files in the PostgreSQL log directory. Below is the configuration, followed by an explanation of each parameter and the reasoning for its inclusion.
{{ pg_log_dir }}/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 {{ postgres_user }} {{ postgres_group }}
    postrotate
        /usr/bin/pkill -u {{ postgres_user }} -HUP || true
    endscript
}

Parameters and Rationale
{{ pg_log_dir }}/*.log

Purpose: Specifies the log files to be rotated.
Details:
{{ pg_log_dir }} is a variable resolving to the PostgreSQL log directory (e.g., /data/test/postgres/15/pg_dbtest_anil/log/).
*.log matches all files with the .log extension in that directory (e.g., postgresql-2025-04-28.log).


Why I Used This:
Ensures the configuration targets only PostgreSQL log files in the designated directory.
The use of a variable makes the setup dynamic, allowing it to adapt to different environments without modifying the playbook.



daily

Purpose: Rotates log files every day.
Details:
A new log file is created daily, and the previous day’s log is rotated.
For example, on April 28, 2025, postgresql-2025-04-27.log is rotated, and a new postgresql-2025-04-28.log is created.


Why I Used This:
Daily rotation ensures log files are manageable in size, making them easier to analyze and preventing them from growing too large.
It aligns with common logging practices for databases, where daily logs are often preferred for troubleshooting and auditing.



rotate 7

Purpose: Retains 7 rotated log files before deleting the oldest.
Details:
Keeps the last 7 rotated files (e.g., postgresql-2025-04-28.log.1.gz to postgresql-2025-04-22.log.7.gz).
On April 29, 2025, the file from April 21 (postgresql-2025-04-21.log.8.gz) would be deleted.


Why I Used This:
Limits disk space usage by retaining only 7 days of logs, which is sufficient for most troubleshooting needs.
Balances the need for historical logs with the risk of filling up disk space.



compress

Purpose: Compresses rotated log files to save disk space.
Details:
Uses gzip to compress rotated files (e.g., postgresql-2025-04-27.log.1 becomes postgresql-2025-04-27.log.1.gz).


Why I Used This:
Reduces the disk space required for archived logs, which is critical in environments with limited storage.
Compression is a standard practice for log management, as logs are highly compressible.



delaycompress

Purpose: Delays compression of the most recently rotated file until the next rotation cycle.
Details:
The most recent rotated file (e.g., postgresql-2025-04-28.log.1) remains uncompressed until the next rotation, when it’s compressed to postgresql-2025-04-28.log.1.gz.


Why I Used This:
Keeps the most recent rotated log uncompressed, making it easier for scripts or tools to read it without needing to decompress.
Useful if another process (e.g., a monitoring tool) needs to access the most recent rotated log immediately after rotation.



missingok

Purpose: Prevents logrotate from failing if no log files exist.
Details:
If {{ pg_log_dir }} contains no .log files, logrotate proceeds without error.


Why I Used This:
Ensures the rotation process is robust and doesn’t fail if PostgreSQL hasn’t yet created log files (e.g., after a fresh setup or if logging is disabled).
Prevents unnecessary errors in the logrotate cron job.



notifempty

Purpose: Skips rotation if the log file is empty (0 bytes).
Details:
If postgresql-2025-04-28.log is empty, logrotate won’t rotate it, even if the daily condition is met.


Why I Used This:
Avoids creating unnecessary rotated files for empty logs, saving disk space and processing time.
Useful in scenarios where PostgreSQL might not write logs on a given day (e.g., low activity or downtime).



create 0640 {{ postgres_user }} {{ postgres_group }}

Purpose: Creates a new log file after rotation with specified permissions and ownership.
Details:
0640: Sets permissions to rw-r----- (read/write for owner, read for group, no permissions for others).
{{ postgres_user }} (e.g., fapgss6283): The owner of the new log file.
{{ postgres_group }} (e.g., postgres): The group of the new log file.


Why I Used This:
Ensures the new log file has secure permissions, restricting access to the PostgreSQL user and group.
Maintains consistency with PostgreSQL’s security model, where log files should only be accessible to the database user and group.



postrotate ... endscript

Purpose: Executes a script after the log file is rotated.
Details:
/usr/bin/pkill -u {{ postgres_user }} -HUP || true:
Sends a SIGHUP signal to PostgreSQL processes owned by {{ postgres_user }} (e.g., fapgss6283).
SIGHUP prompts PostgreSQL to reopen its log file, ensuring it writes to the new file after rotation.
|| true ensures logrotate doesn’t fail if pkill doesn’t find any processes.




Why I Used This:
PostgreSQL keeps log files open while running. Without this, it might continue writing to the old (rotated) file.
The SIGHUP signal ensures a seamless transition to the new log file, preventing log data loss or inconsistency.



Why This Configuration?
This logrotate configuration was chosen to balance several key considerations:

Efficiency: Daily rotation (daily) and compression (compress, delaycompress) keep log files manageable and reduce disk space usage.
Reliability: Parameters like missingok and notifempty make the setup robust, handling edge cases (e.g., missing or empty logs) without errors.
Security: The create 0640 directive ensures new log files have secure permissions, limiting access to the PostgreSQL user and group.
Operational Continuity: The postrotate script ensures PostgreSQL continues logging to the new file after rotation, avoiding disruptions.
Disk Space Management: rotate 7 limits retention to 7 days, preventing excessive disk usage while retaining enough history for troubleshooting.

Deployment Instructions

Setup Directory Structure:mkdir -p logrotate_config_deployment


Ansible Playbooks:
Ensure pgsql-06-logrotate-config.yml and logrotate_config.yml are in the correct locations (as shown in the file structure).


Run the Playbook:ansible-playbook -l hostname pgsql-06-logrotate-config.yml


Replace hostname with the target host (e.g., s1t1dpgs01std).
Add -i <inventory_file> if using a custom inventory.
Add --become-pass if sudo requires a password.



Testing the Configuration

Verify the Configuration File:cat /etc/logrotate.d/postgresql


Confirm the configuration matches the expected settings.


Dry-Run Logrotate:/usr/sbin/logrotate -d /etc/logrotate.d/postgresql


Ensures no syntax errors and shows what logrotate will do.


Force Rotation:/usr/sbin/logrotate -f /etc/logrotate.d/postgresql


Manually rotates the logs to verify behavior.


Check Permissions:ls -l /data/test/postgres/15/pg_dbtest_anil/log/


Ensure log files are owned by fapgss6283:postgres with 0640 permissions.



Troubleshooting

No Log Files Exist:
The missingok parameter ensures logrotate doesn’t fail.
Verify PostgreSQL is configured to write logs to {{ pg_log_dir }} (check log_directory in postgresql.conf).


PostgreSQL Still Writes to Old Log:
Confirm the postrotate script is working by checking if pkill sends the SIGHUP signal.
Ensure /usr/bin/pkill exists on the target host.


Disk Space Issues:
If disk space fills up, consider reducing rotate 7 to a smaller value (e.g., rotate 5) or adding size 300M to rotate based on file size.



Future Enhancements

Size-Based Rotation: Add size 300M to rotate logs if they exceed 300 MB, preventing large files within a single day.
Age-Based Retention: Add maxage 7 to delete rotated files older than 7 days, complementing rotate 7.
Archive Directory: Use olddir to move rotated logs to a separate directory for better organization.

Author

Name: Anilkumar Sonawane
Date: April 28, 2025

