# PostgreSQL Cluster Backup With pgBackRest

In today's lecture, we will be exploring how to backup our PostgreSQL database using PgBackRest. My name is Ilori Stephen, and I am a senior software engineer with over 5 years of experience in building systems at scale.

If you're someone like me who loves to live on the edge and prefers managing their own databases over managed services, then you've come to the right place. If you also love to play it safe and use managed services, then we can still learn a thing or two on backing up our lovely PostgreSQL databases.

## What Is PgBackRest?

PgBackRest is a PITR (Point In Time Recovery) solution for PostgreSQL. It takes advantage of WAL Archiving to backup and restore our database. It is a reliable tool that comes built in with many features to help us backup and restore databases at scale.

Some of its key features include:

- **Full, Differential & Incremental Backups:** We can perform differential and incremental backups at block level (only copying parts of a file that have changed).
- **Compression:** With compression algorithms like Zstd and lz4, we can save ourselves some time when backing up large databases, although at the cost of CPU.
- **Retention Policies:** Backups eat space. With PgBackRest, we can control how many backups we want to keep.
- **Parallel Async WAL Push & Get:** The commands for pushing and retrieving WAL segments are built to support parallelism and run asynchronously.
- **S3, Azure & GCS Support:** This won me over. The ability to backup my databases to a dedicated blob storage is critical for me, because these can grow to support the size of my backups and give me different restore options.

## Installation

For the purpose of this article, I will focus on Debian & Ubuntu systems. We will start by installing PgBackRest from the package stores by running this command (or your package manager equivalent).

```bash
sudo apt-get install pgbackrest
```

This is preferred to building from source as most of the configuration and data directories would have been initialized. ***It is very important that you have PgBackRest installed on the same machine as your PostgreSQL server.***

To confirm that the installation worked, we can verify by running this command:

```bash
sudo -u postgres pgbackrest
```

We should get the following output:

```
pgBackRest 2.58.0 - General help

Usage:
pgbackrest [options] [command]

Commands:
annotate        add or modify backup annotation
archive-get     get a WAL segment from the archive
archive-push    push a WAL segment to the archive
backup          backup a database cluster
check           check the configuration
expire          expire backups that exceed retention
help            get help
info            retrieve information about backups
repo-get        get a file from a repository
repo-ls         list files in a repository
restore         restore a database cluster
server          pgBackRest server
server-ping     ping pgBackRest server
stanza-create   create the required stanza data
stanza-delete   delete a stanza
stanza-upgrade  upgrade a stanza
start           allow pgBackRest processes to run
stop            stop pgBackRest processes from running
verify          verify contents of a repository
version         get version

Use 'pgbackrest help [command]' for more information.
```

Now that we can confirm PgBackRest is installed and running, we can move on to the next step.

## Backing Up Our Database

In order to backup our database, we will need to create a PgBackRest configuration file. The configuration file has a number of options, and I will try to explain them as best as I can:

- **Stanza:** A stanza in PgBackRest defines where our database cluster is located, how it will be backed up, and the different archiving options. We will create one in a bit.
- **Repository:** The repository is where PgBackRest stores our backups and WAL segments. A default repository is created during installation at ***/var/lib/pgbackrest***. You can also provide a custom repository path, but that is outside the scope of this article.
- **PostgreSQL Archiving:** In order to backup our PostgreSQL cluster, we need to set up WAL archiving. PostgreSQL generates a consistent **16MB WAL segment**. **WAL** generation is always on by default, but we need to modify the archive command to use our PgBackRest stanza.

We will go over these steps in more detail, and by the end we should have our database backed up. ***You should probably do a pg_dump on your database if you haven't done so already, so we have a safe point to restore to during this lecture should the need arise.***

### Creating The Stanza

This is the first step to backing up our database cluster. To get started, we need to create a configuration file. I prefer creating mine in the `/etc/pgbackrest` directory. The `/etc` folder already exists, so we are simply creating a `pgbackrest` folder inside it.

```bash
cd /etc && sudo mkdir pgbackrest && cd pgbackrest && sudo touch [desired-name].conf
```

The command above creates our pgbackrest configuration folder and our stanza configuration file. The next step is setting the right permissions so we can access our stanza when required.

```bash
sudo chmod 640 /etc/pgbackrest/[stanza-name].conf
sudo chown postgres:postgres /etc/pgbackrest/[stanza-name].conf
```

With the steps above done, we should have our configuration file initialized. The next step is populating it with options. Below is my preferred setup for backing up a database. I always go with the dual repository option — backing up to the local data directory and to an AWS S3 bucket.

```ini
[global]
# Local repository
repo1-path=/var/lib/pgbackrest                # Where backups and WAL archives are stored on disk
repo1-retention-full=2                        # Keep the last 2 full backups (older ones are expired)
repo1-bundle=y                                # Bundle small files together to reduce I/O overhead
repo1-block=y                                 # Enable block-level incremental (only copy changed blocks, not entire files)

# S3 repository
repo2-type=s3                                 # Use Amazon S3 as the storage backend
repo2-s3-bucket=[bucket-name]                 # The name of your S3 bucket
repo2-s3-region=[region]                      # AWS region where the bucket is hosted
repo2-s3-endpoint=s3.[region].amazonaws.com   # S3 endpoint URL (adjust for non-AWS S3-compatible stores)
repo2-s3-key=                                 # AWS access key ID (leave empty if using IAM roles)
repo2-s3-key-secret=                          # AWS secret access key (leave empty if using IAM roles)
repo2-path=/pgbackrest                        # Path prefix inside the S3 bucket for storing backups
repo2-retention-full=7                        # Keep the last 7 full backups on S3 (longer retention than local)
repo2-bundle=y                                # Bundle small files together for S3 (reduces API calls)
repo2-block=y                                 # Block-level incremental for S3 (reduces transfer size)

# S3 timeout settings
repo2-storage-upload-chunk-size=5242880       # 5MB upload chunks (default 1MB; larger chunks = fewer requests)
io-timeout=300000                             # 5 minute I/O timeout in ms (default 60s; increase for large backups)

# Global settings
compress-type=zst                             # Use Zstandard compression (fast speed, good ratio)
process-max=2                                 # Max parallel processes for backup/restore operations
log-path=/var/log/pgbackrest                  # Directory for pgBackRest log files
lock-path=/var/lock/pgbackrest                # Directory for lock files (prevents concurrent operations)
```

For my use-case, this setup works best. PgBackRest supports more than one repository — for each new repository you want to add, simply suffix `repo` with a version number (`2` in the case of S3 being our secondary option).

With our stanza now set up, the next thing we need to do is modify our PostgreSQL cluster archive configuration.

### PostgreSQL Archiving

This is the second critical part of our setup. In order to backup our database, we need to enable WAL archiving from our cluster configuration. Our PostgreSQL database forces a WAL switch periodically, and we need to configure the archive command to use PgBackRest.

It is important to note that enabling this will cause at least one WAL file to be generated. We need to edit the following file to get started:

```bash
sudo nano /etc/postgresql/17/main/postgresql.conf
```

Scroll (or search) to the **Archiving** section and enter the following configuration:

```ini
archive_mode = on
archive_command = 'pgbackrest --config=/etc/pgbackrest/[stanza-name].conf --stanza=[stanza-name] archive-push %p'
```

With this in place, we need to restart our PostgreSQL cluster for the changes to take effect:

```bash
sudo service postgresql restart
```

There should be no errors when we run this command, assuming everything is configured correctly.

### Testing

Our next step is to create our stanza, check our configuration, and perform a backup.

**Create the stanza:**

```bash
sudo -u postgres pgbackrest --config=/etc/pgbackrest/[stanza-name].conf --stanza=[stanza-name] stanza-create
```

This command will initialize our stanza on both repositories. After running it successfully, we can verify our configuration:

**Check the configuration:**

```bash
sudo -u postgres pgbackrest --config=/etc/pgbackrest/[stanza-name].conf --stanza=[stanza-name] check
```

If this completes with no errors, we are good to go. Now we can create our first backup. Depending on the size of our database, this can take a while — for that reason I always recommend increasing the `archive-timeout` option in the PgBackRest configuration.

**Perform a full backup:**

```bash
sudo -u postgres pgbackrest --config=/etc/pgbackrest/[stanza-name].conf --stanza=[stanza-name] backup --type=full
```

This command will take a full backup of our database. Once complete, we can experiment with the different backup types. To take an incremental or differential backup, simply change the value passed to the `--type` flag:

```bash
# Differential backup (changes since last full backup)
sudo -u postgres pgbackrest --config=/etc/pgbackrest/[stanza-name].conf --stanza=[stanza-name] backup --type=diff

# Incremental backup (changes since last backup of any type)
sudo -u postgres pgbackrest --config=/etc/pgbackrest/[stanza-name].conf --stanza=[stanza-name] backup --type=incr
```

We can also verify our backup info at any time:

```bash
sudo -u postgres pgbackrest --config=/etc/pgbackrest/[stanza-name].conf --stanza=[stanza-name] info
```

## Security & Encryption

When backing up databases, especially to remote storage like S3, encryption should be a priority. PgBackRest supports repository encryption out of the box using AES-256-CBC, which encrypts both the backup files and the WAL archive at rest.

To enable encryption, add the following to the `[global]` section of your stanza configuration:

```ini
[global]
# Encrypt the local repository
repo1-cipher-type=aes-256-cbc
repo1-cipher-pass=[your-encryption-passphrase]

# Encrypt the S3 repository
repo2-cipher-type=aes-256-cbc
repo2-cipher-pass=[your-encryption-passphrase]
```

A few important notes on encryption:

- **Store your passphrase securely.** If you lose the passphrase, your backups become unrecoverable. Consider using a secrets manager or a secure vault to store it.
- **Encryption must be set before creating the stanza.** You cannot enable encryption on an existing repository. If you need to add encryption later, you will need to create a new stanza and take a fresh full backup.
- **Use different passphrases per repository** if you want to further isolate access between your local and remote backups.
- **S3 server-side encryption (SSE)** can be used alongside PgBackRest encryption for defense in depth. You can enable SSE on your S3 bucket independently through AWS configuration.

Beyond encryption, here are a few more security practices to consider:

- **Restrict file permissions** on your configuration files since they contain sensitive credentials. As we configured earlier, `chmod 640` with `postgres:postgres` ownership ensures only the PostgreSQL user and root can read them.
- **Use IAM roles** instead of hardcoded S3 keys when running on AWS EC2. PgBackRest supports IAM role-based authentication by omitting the `repo-s3-key` and `repo-s3-key-secret` options.
- **Regularly verify your backups** using `pgbackrest verify` to ensure backup integrity has not been compromised.

## Scheduling Backups with Cron

Running backups manually is fine for testing, but in production we want these to run automatically. We can use cron to schedule our backups at regular intervals.

Edit the crontab for the `postgres` user:

```bash
sudo crontab -u postgres -e
```

Here is an example schedule that works well for most use-cases:

```cron
# Full backup every Sunday at 1:00 AM
0 1 * * 0 pgbackrest --config=/etc/pgbackrest/[stanza-name].conf --stanza=[stanza-name] backup --type=full >> /var/log/pgbackrest/cron.log 2>&1

# Differential backup every day (except Sunday) at 1:00 AM
0 1 * * 1-6 pgbackrest --config=/etc/pgbackrest/[stanza-name].conf --stanza=[stanza-name] backup --type=diff >> /var/log/pgbackrest/cron.log 2>&1
```

Note that we don't need a separate `expire` cron job — since PgBackRest 2.x, the `expire` command runs automatically after every backup. It cleans up old backups and WAL segments based on the `repo-retention-full` value we set in our configuration.

We redirect output to a log file so we have visibility into any failures. PgBackRest also writes its own detailed logs (configured via `log-level-file` in our stanza) to `/var/log/pgbackrest/` by default.

## Conclusion

PgBackRest is one of the most reliable backup tools available for PostgreSQL, and having gone through the process of setting it up myself, I can confidently say it is worth the effort. The combination of block-level incremental backups, built-in compression, and native support for cloud storage like S3 makes it a solid choice for anyone managing their own PostgreSQL infrastructure.

To recap what we covered:

1. **Installation** — Getting PgBackRest set up alongside our PostgreSQL server.
2. **Stanza Configuration** — Defining our backup repositories (local and S3) and cluster path.
3. **WAL Archiving** — Configuring PostgreSQL to push WAL segments through PgBackRest.
4. **Testing** — Creating the stanza, verifying the configuration, and running our first full backup.
5. **Security** — Enabling AES-256 encryption and following best practices for credential management.
6. **Scheduling** — Automating backups with cron to ensure consistent protection.

If you are managing your own PostgreSQL databases, I strongly recommend investing the time to set this up. The peace of mind that comes with knowing your data is backed up, encrypted, and recoverable is well worth it. And if you are using managed services, understanding how backup and recovery works under the hood makes you a better engineer when things inevitably go sideways.

If you have any questions or run into issues following along, feel free to reach out. Happy hacking lol!
