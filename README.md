### OpenObserve Setup Summary

OpenObserve is a log and metrics analysis platform. MinIO is used for data storage and PostgreSQL as the metadata store. PostgreSQL backups are created with `pg_dump` and saved to `MinIO` for safekeeping as cron jobs.

---

### Inventory (`inventory/hosts.ini`)

Remember to change the following:
```ini
ansible_host=<ip_of_target_host>
ansible_ssh_private_key_file=<path/of/generated/priv_key>
```

---

### Playbook Execution

Run all plays at once:

```bash
ansible-playbook playbooks/all.yml
```

Or run sequentially:

1. `playbooks/system_prep.yml`: runs role `system_prep`
2. `playbooks/docker_install.yml`: runs role `docker_install`
3. `playbooks/postgres_docker.yml`: runs role `postgres_docker`
4. `playbooks/openobserve_docker.yml`: runs role `openobserve_docker`
5. `playbooks/beszel_agent.yml`: runs role `beszel_agent`


### `roles/system_prep`

#### vars:
- `dev_password`: Set secure password for dev user
- `pub_key`: Generated public key to be stored in `~/.ssh/authorized_keys` for SSH access

#### tasks:
1. Verifies the system runs Debian; aborts otherwise.
2. Installs essential system packages.
3. Generates and sets the system locale to `en_US.UTF-8`.
4. Sets the system timezone to UTC.
5. Creates a new user `dev` with a secure password and home directory.
6. Grants `dev` sudo privileges.
7. Disables root shell access and root login over SSH.
8. Changes the SSH port from default `22` to `10822` for security.
9. Adds the provided SSH public key for the `dev` user.
10. Allows SSH connections on the new custom port (10822).
11. Installs and configures UFW firewall: denies all incoming, allows outgoing connections.
12. Enables the firewall with these rules active.

### `roles/docker_install`

#### tasks:
This sequence installs Docker on Debian by setting up Docker’s GPG key and repository, installing Docker packages, creating the docker group, adding the `dev` user to it, starting the Docker service, and verifying the installation.


### `roles/postgres_docker`

#### vars:

* **container\_name**: Name assigned to the PostgreSQL Docker container.
* **postgres\_user**: Username for the PostgreSQL database.
* **postgres\_password**: Password for the PostgreSQL user.
* **postgres\_db**: Name of the PostgreSQL database to use or create.
* **minio\_alias**: Reference name used for MinIO in configurations or CLI.
* **minio\_url**: Endpoint URL for accessing the MinIO server.
* **minio\_access\_key**: Access key for authenticating with MinIO.
* **minio\_secret\_key**: Secret key for authenticating with MinIO.
* **minio\_bucket**: Bucket name in MinIO used to store data.

#### tasks:
1. Pulls the PostgreSQL Docker image version 17.
2. Creates a Docker volume for persistent PostgreSQL data.
3. Creates a Docker network for OpenObserve and PostgreSQL containers.
4. Runs the PostgreSQL container with environment variables, volume, network, and port mapping.
5. Creates a backup directory in the `dev` user’s home folder.
6. Downloads and installs the MinIO client (`mc`).
7. Adds MinIO client path to the `dev` user’s environment.
8. Creates MinIO configuration directory for the `dev` user.
9. Writes MinIO access configuration for the `dev` user.
10. Adds a backup script for PostgreSQL that dumps the database, compresses it, uploads it to MinIO, and cleans old backups.
11. Sets a daily cron job to run the backup script at 2 AM as the `dev` user.


### `roles/openobserve_docker`

#### vars:

* **root\_email**: Admin email address for notifications or user setup.
* **root\_password**: Password for the root or admin user.
* **postgres\_user**: PostgreSQL database username.
* **postgres\_password**: Password for the PostgreSQL user.
* **s3\_address**: URL endpoint for the MinIO (S3-compatible) storage.
* **s3\_access\_key**: Access key ID for MinIO authentication.
* **s3\_secret\_key**: Secret access key for MinIO authentication.
* **s3\_bucket\_name**: Name of the storage bucket in MinIO for data storage.

#### tasks:
1. Pulls the OpenObserve Docker image version 0.10.9.
2. Creates a log directory for OpenObserve with proper ownership and permissions.
3. Runs the OpenObserve container with environment variables for user, metadata (Postgres), and storage (MinIO), exposes port 5080, mounts log directory, and connects to the Docker network.

### `roles/beszel_agent`

#### tasks:

1. Starts and ensures the Beszel agent container runs continuously.
2. You need to register this system in the Beszel hub to enable monitoring and management.
