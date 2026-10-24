# Linux 

This is a local, production-grade Linux environment running Ubuntu 26.04. It is secured with SSH key authentication and managed via Docker Compose, mimicking a real-world cloud VPS.

## Directory Structure

* `Dockerfile`: Contains the recipe for the Ubuntu image, tools, and SSH security setup.
* `docker-compose.yml`: Manages the infrastructure, port mapping (2222->22), and volume mounts.
* `scripts/`: Local folder synced to `/workspace/scripts` inside the container.
* `data/`: Local folder synced to `/workspace/data` inside the container.
* `lab_key` & `lab_key.pub`: Your private and public SSH keys (Private key is ignored by Git).

---

## Quick Start Guide

### 1. Generate SSH Keys (One-time setup)
Before building the container for the first time, generate the SSH key pair in this folder:
```bash
ssh-keygen -t ed25519 -f ./lab_key
```

### 2. How to run
```bash
docker-compose up -d --build

# ssh-keygen -R [localhost]:2222 
# SH to forget the old container's identity: 

ssh -i ./lab_key devops@localhost -p 2222
```
# for change the  roles then 
```bash
 sudo ls -la /home/test/ 

 sudo su -test
```
then use Exit 