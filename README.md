# Deployment of SPO Data Processing Pipeline

This project deploys a data processing pipeline for **Smartphone Pressure Observations (SPO)** to a virtual machine using **Ansible** and **Docker**.

The pipeline retrieves pressure observations from **Firebase**, processes them, and:

- Sends observations to **ESOH**
- Stores raw JSON data in **ECMWF Object Storage (S3-compatible)**
- Deletes processed data from **Firebase after successful storage**

The pipeline is executed inside **Docker containers** and scheduled via **cron** to run periodically.

---

# Architecture Overview

The deployed pipeline performs the following steps:

1. Fetch SPO observations from **Firebase**
2. Convert observations into **ESOH-compatible features**
3. Upload features to **ESOH**
4. Store raw observation data in **ECMWF Object Storage**
5. Delete processed data from Firebase after successful storage

If ESOH upload fails, the pipeline retries up to **5 times** before continuing.  
Firebase deletion depends on **successful storage in object storage**, ensuring that no data is lost.

---

# Deployment Requirements

The deployment requires:

- An **EWC VM**
- **Docker** installed on the VM (handled by Ansible)
- **Ansible** installed locally
- **SSH access** to the VM
- **Firebase credential files**
- **Object storage credentials**
- **ESOH credentials**

---

# Repository Structure

```
ansible/
 ├── ewc.yml
 ├── inventory
 └── roles/
      └── spo-pipeline/
          ├── tasks/
          │    └── main.yaml
          └── vars/
               ├── main.yml
               └── credentials.yml

spo_firebase_fetcher/
 ├── fetcher.py
 └── firebase_config.py

Dockerfile
pyproject.toml
README.md
```

---

# Configuration

## Main configuration

`ansible/roles/spo-pipeline/vars/main.yml`

Example:

```yaml
fetcher_image: "your-registry/spo-firebase-fetcher:latest"

fetcher_apps:
  - dmi
  - sfs

spo_base_path: "/opt/spo"

object_store_endpoint: "https://object-store.os-api.cci1.ecmwf.int"
object_store_bucket: "spo-firebase-data"
object_store_region: "default"
```

---

## Credentials

Credentials should be stored in:

```
ansible/roles/spo-pipeline/vars/credentials.yml
```

Example:

```yaml
object_store_access_key: "YOUR_ACCESS_KEY"
object_store_secret_key: "YOUR_SECRET_KEY"

esoh_username: "YOUR_ESOH_USERNAME"
esoh_password: "YOUR_ESOH_PASSWORD"
```

For security, this file should be encrypted using **Ansible Vault**.

Example:

```bash
ansible-vault encrypt ansible/roles/spo-pipeline/vars/credentials.yml
```

---

# Firebase Credential Files

The deployment expects Firebase credential files on the VM:

```
/opt/spo/dmiapp_firebase.json
/opt/spo/sfsapp_firebase.json
```

These files are mounted inside the container as:

```
/creds/credentials.json
```

---

# Docker Image

The Docker image runs the SPO fetcher:

```
python -m spo_firebase_fetcher.fetcher
```

The container receives configuration through environment variables provided by Ansible.

---

# Deploying the Pipeline

To deploy the pipeline to the VM:

```bash
uv run ansible-playbook \
  ansible/ewc.yml \
  --inventory ansible/inventory \
  --user <ewc_user> \
  --private-key <private_key> \
  --become
```

Example:

```bash
uv run ansible-playbook \
  ansible/ewc.yml \
  --inventory ansible/inventory \
  --user elbadmin \
  --private-key ~/.ssh/id_ewc \
  --become
```

If sudo requires a password, include:

```bash
--ask-become-pass
```

---

# What the Playbook Does

The playbook performs the following actions:

1. Installs Docker on the VM
2. Pulls the SPO fetcher Docker image
3. Runs one container per SPO app (`dmi`, `sfs`)
4. Mounts Firebase credential files
5. Injects object storage and ESOH credentials
6. Configures cron jobs to run the fetcher every 10 minutes

---

# Running Containers

After deployment, the following containers will run:

```
firebase-fetcher-dmi
firebase-fetcher-sfs
```

---

# Monitoring the Pipeline

Check running containers:

```bash
docker ps
```

Check logs:

```bash
docker logs firebase-fetcher-dmi
```

Example log output:

```
Sending 42 feature(s) to ESOH
ESOH upload successful
Uploaded spo_data_dmi_20240601T120000.json
Deleted data from Firebase
```

---

# Cron Scheduling

The pipeline runs every **10 minutes** via cron.

Example cron entry:

```
*/10 * * * * docker restart firebase-fetcher-dmi
```

---

# Local Testing

To run the fetcher locally without object storage:

```bash
pdm run python -m spo_firebase_fetcher.fetcher \
  --cred_path path/to/firebase.json \
  --app dmi \
  --time_limit 1 \
  --no_s3
```

---

# Data Safety

The pipeline is designed to prevent data loss:

- ESOH upload is retried **5 times**
- Raw data is stored in object storage
- Firebase data is deleted **only after successful storage**

---

# Troubleshooting

### Check container logs

```bash
docker logs firebase-fetcher-dmi
```

### Verify object storage connectivity

Check that environment variables are present inside the container:

```bash
docker inspect firebase-fetcher-dmi
```

### Verify Firebase credential mounting

```bash
docker exec -it firebase-fetcher-dmi ls /creds
```

---
