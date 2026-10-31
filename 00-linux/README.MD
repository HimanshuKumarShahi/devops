#  Complete Linux & DevOps Local Lab Guide (Ubuntu 26.04)

This is a local, production-grade Linux environment running Ubuntu 26.04. It is secured with SSH key authentication and managed via Docker Compose, mimicking a real-world cloud VPS (like AWS EC2 or DigitalOcean). 

This single document contains **all project infrastructure code**, **setup steps**, **user & group administration**, **permission models**, and **troubleshooting fixes** for self-study.

---

## 📁 1. Directory Structure

- **`Dockerfile`**: Contains the recipe for the Ubuntu image, tools, user setup, and SSH security.
- **`docker-compose.yml`**: Manages the infrastructure, port mapping (`2222->22`), and volume mounts.
- **`scripts/`**: Local folder synced to `/workspace/scripts` inside the container.
- **`data/`**: Local folder synced to `/workspace/data` inside the container.
- **`lab_key` & `lab_key.pub`**: Your private and public SSH keys (Keep `lab_key` in `.gitignore`).

---

## ⚙️ 2. Complete Infrastructure Code

### A. `Dockerfile`
Create a file named `Dockerfile` (no extension) in your project folder and paste this exact code:

```dockerfile
FROM ubuntu:26.04

ENV DEBIAN_FRONTEND=noninteractive

# Install essential DevOps and networking tools
RUN apt-get update && apt-get install -y \
    sudo \
    curl \
    wget \
    vim \
    nano \
    git \
    iproute2 \
    iputils-ping \
    net-tools \
    htop \
    tree \
    openssh-server \
    && rm -rf /var/lib/apt/lists/*

# Create SSH run directory and generate host keys so sshd doesn't crash
RUN mkdir -p /var/run/sshd && ssh-keygen -A

# Create devops user with root primary group (-g root) and sudo privileges (-G sudo)
RUN useradd -rm -d /home/devops -s /bin/bash -g root -G sudo devops

# Set password for sudo authentication prompts
RUN echo "devops:ubuntu123" | chpasswd

# Optional: Uncomment the line below if you ever want passwordless sudo
# RUN echo "devops ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/90-devops

# Setup SSH key authentication for devops user
RUN mkdir -p /home/devops/.ssh
COPY lab_key.pub /home/devops/.ssh/authorized_keys

# Apply strict SSH permissions (SSH refuses to work if permissions are too open)
RUN chown -R devops:root /home/devops/.ssh \
    && chmod 700 /home/devops/.ssh \
    && chmod 600 /home/devops/.ssh/authorized_keys

# Configure SSHD to allow Public Key Auth and disable insecure password logins
RUN sed -i 's/#PubkeyAuthentication yes/PubkeyAuthentication yes/' /etc/ssh/sshd_config \
    && sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config

# Setup shared workspace and give the devops user ownership
WORKDIR /workspace
RUN chown -R devops:root /workspace

EXPOSE 22

CMD ["/usr/sbin/sshd", "-D"]
```

### B. `docker-compose.yml`
Create a file named `docker-compose.yml` in the same folder and paste this exact code:

```yaml
services:
  linux:
    build: 
      context: .
    container_name: linux
    ports:
      - "2222:22"
    volumes:
      - ./scripts:/workspace/scripts
      - ./data:/workspace/data
```

---

## 🚀 3. Quick Start Guide (How to Run)

```bash
# 1. Generate SSH Keys (One-time setup in your local project folder)
ssh-keygen -t ed25519 -f ./lab_key

# 2. Build and start the container in the background
docker-compose up -d --build

# 3. Clear old container SSH fingerprint (Run ONLY if you rebuilt the container and get an error)
ssh-keygen -R [localhost]:2222 

# 4. Connect to the Linux container via SSH
ssh -i ./lab_key devops@localhost -p 2222
```

**Why we need each step:**
* **`ssh-keygen -t ed25519 -f ./lab_key`**: Generates a modern, cryptographic key pair (`lab_key` is your secret private key; `lab_key.pub` is copied into the server).
* **`docker-compose up -d --build`**: Reads `docker-compose.yml` and builds the `Dockerfile`. `-d` runs it in the background. *(Note: If you get `no configuration file provided: not found`, make sure your terminal is inside the folder where `docker-compose.yml` is saved).*
* **`ssh-keygen -R [localhost]:2222`**: Every time you destroy and rebuild the container, it gets a new identity fingerprint. This command tells your computer to forget the old container's identity so SSH doesn't block you.
* **`ssh -i ./lab_key devops@localhost -p 2222`**: Connects to your container as the `devops` user using your private key (`-i`) over custom port `2222` (`-p`).

---

## 👑 4. Admin Roles & Switching Users (`sudo` & `su`)

In Linux, `root` is the all-powerful administrator, and `devops` is your daily operator. By default, every user's home directory (`/home/ubuntu`, `/home/test`) is locked to protect privacy. Even as `devops`, running `cd /home/ubuntu` gives `-bash: cd: ubuntu/: Permission denied`.

To perform admin tasks or bypass locks, you must explicitly use `sudo` (which asks for your password: `ubuntu123`).

```bash
# 1. Peek inside a locked home folder without switching users
sudo ls -la /home/test/

# 2. Switch your identity to another user (e.g., test)
sudo su - test

# 3. Switch your identity to the ultimate root admin
sudo su -

# 4. Return to your main devops account
exit
```

**How `sudo su - test` works:**
* **`sudo`**: Borrow admin privileges so you don't have to know or type the `test` user's password.
* **`su`**: Switch User.
* **`-`**: Tells Linux to completely load the new user's environment and teleport you directly into their home folder (`/home/test`), just like a fresh login. *(Always keep a space between `-` and the username! `sudo su -test` will fail).*
* **`test`**: The exact username you want to become.
* **`exit`**: Because `su` opens a temporary session *inside* your original terminal, typing `exit` (or pressing `Ctrl + D`) closes the `test` session and wakes up your `devops` user. *(Note: Standard users like `test` are not in the `sudo` group. If `test` tries to run `sudo`, Linux prints: `sudo: I'm sorry test. I'm afraid I can't do that`).*

---

## 👥 5. User & Group Management (Step-by-Step)

### A. Creating Users (`adduser` vs `useradd`)

```bash
# Method 1: The Interactive & Friendly Way (Recommended)
sudo adduser himanshu

# Method 2: The Basic Low-Level Way
sudo useradd testuser

# Delete a user and their home directory completely
sudo userdel -r testuser
```

**Key Difference:**
* **`adduser himanshu`**: Automatically creates `/home/himanshu`, assigns the colorful `/bin/bash` shell, and prompts you to set a password and full name.
* **`useradd testuser`**: If run without flags, it assigns the basic `/bin/sh` shell (which lacks arrow-key history and colors) and might not create a home folder unless you add `-m -s /bin/bash`.

### B. Creating Groups & Adding Users (`groupadd` & `usermod`)

Groups do not have home directories (you will not see them when running `ls /home`). They are security labels stored in `/etc/group` so multiple users can share folders.

```bash
# 1. Create a new group
sudo groupadd developers

# 2. Add an existing user (himanshu) to the group (developers)
sudo usermod -aG developers himanshu

# 3. Quick check to see all IDs and groups a user belongs to
id himanshu
```

**How `sudo usermod -aG developers himanshu` works:**
* **`usermod`**: Modify an existing user account.
* **`-a` (Append)**: **CRITICAL.** Tells Linux to *add* the user to the new group without removing them from their existing groups.
* **`-G` (Groups)**: Specifies the secondary group name (`developers`).

---

## 🔍 6. Inspecting System Databases (`/etc/passwd`, `/etc/group`, `grep`)

**⚠️ Critical `grep` Warning:** Never type `grep himanshu` by itself without a file path! If you forget the file name, `grep` waits forever for keyboard input and freezes your terminal. Press **`Ctrl + C`** to cancel and escape.

### A. Checking User Accounts (`/etc/passwd`)

```bash
# View all users on the system
cat /etc/passwd

# Search for a specific user in the password database
grep himanshu /etc/passwd
```

**How to read the output:**
`himanshu:x:1004:1004:himanshu kumar,,,:/home/himanshu:/bin/bash` (7 fields separated by `:`)
1. **`himanshu`**: Username.
2. **`x`**: Password is encrypted and stored safely in `/etc/shadow`.
3. **`1004`**: User ID (UID). *(Note: `root` is always `0`).*
4. **`1004`**: Primary Group ID (GID). *(In our Dockerfile, `devops` has `0`, meaning its primary group is `root`).*
5. **`himanshu kumar,,,`**: User description / full name.
6. **`/home/himanshu`**: Home directory path.
7. **`/bin/bash`**: Default login shell. *(System accounts like `www-data` or `sshd` have `/usr/sbin/nologin` here so hackers cannot log into them).*

### B. Checking Groups (`/etc/group`)

```bash
# View all groups on the system
cat /etc/group

# Search for a specific group or see every group a user belongs to
grep developers /etc/group
grep himanshu /etc/group
```

**How to read the output:**
When you ran `grep himanshu /etc/group`, Linux returned three lines:
* `users:x:100:himanshu` -> Member of general `users` group (GID `100`).
* `developers:x:1003:himanshu` -> Group name (`developers`), encrypted password (`x`), GID (`1003`), and group members (`himanshu`).
* `himanshu:x:1004:` -> The user's personal primary group.

---

## 🔒 7. File Permissions & Ownership (`chmod` & `chown`)

When you run `ls -l`, Linux displays a 10-character permission string like `-rwxr-xr--` or `drwxrwx---`.

### A. How to Read `ls -l` Permissions
1. **1st Character (File Type):** `-` means regular file; `d` means Directory (folder).
2. **Characters 2–4 (Owner/User):** Permissions for the person who owns the file.
3. **Characters 5–7 (Group):** Permissions for members of the file's assigned group.
4. **Characters 8–10 (Others):** Permissions for everyone else on the server.

### B. The `chmod` Number System
Each action has a math value. Add them together for **Owner**, **Group**, and **Others**:
* **Read (`r`) = `4`** (View file contents or list folder items)
* **Write (`w`) = `2`** (Edit file or create/delete files in a folder)
* **Execute (`x`) = `1`** (Run a script/program or `cd` into a folder)
* **No Access (`-`) = `0`**

| Command | Math Breakdown | Symbol Output | What It Means |
| :--- | :--- | :--- | :--- |
| `chmod 600 file.txt` | `4+2` / `0` / `0` | `-rw-------` | **Private:** Only the owner can read/write. Nobody else can access. |
| `chmod 644 file.txt` | `4+2` / `4` / `4` | `-rw-r--r--` | **Standard File:** Owner can read/write; everyone else can only read. |
| `chmod 755 script.sh` | `4+2+1` / `4+1` / `4+1` | `-rwxr-xr-x` | **Executable Script:** Owner has full control; others can read and run it. |
| `chmod 770 folder/` | `4+2+1` / `4+2+1` / `0` | `drwxrwx---` | **Team Folder:** Owner and Group members have full access; outsiders blocked. |

### C. Changing Ownership (`chown`) & Sharing Folders
If you copy files using `sudo`, `root` becomes the owner. Use `chown` to change the **User** and **Group** owner (`user:group`).

```bash
# 1. Create a shared project folder for the developers group
sudo mkdir -p /workspace/dev_project

# 2. Give ownership to devops (user) and developers (group) recursively (-R)
sudo chown -R devops:developers /workspace/dev_project

# 3. Give full read/write/execute access (770) to devops and everyone in 'developers' (like himanshu)
sudo chmod -R 770 /workspace/dev_project
```

---

## 📝 8. Writing, Copying, & Deleting Files

### A. 3 Ways to Write to Files

```bash
# Method 1: Quick One-Liner (Single > OVERWRITES the whole file)
echo "This is line 1" > /workspace/scripts/test.txt

# Method 2: Quick Append (Double >> ADDS to the bottom without deleting)
echo "This is line 2" >> /workspace/scripts/test.txt

# Method 3: Terminal Text Editor (Nano)
nano /workspace/scripts/test.txt
# Inside Nano: Type text -> Press Ctrl+O then Enter to Save -> Press Ctrl+X to Exit
```
*(**Method 4 - The DevOps Way:** Open the `scripts/` or `data/` folder directly in VS Code on your host machine, edit, and save. It syncs instantly into `/workspace`!).*

### B. Copying & Deleting Troubleshooting

```bash
# Copy an entire directory and all locked user folders inside it
sudo cp -r /home /workspace/scripts/

# Force delete a write-protected folder without confirmation prompts
sudo rm -rf folder_name
```

**Why your earlier commands failed:**
1. **`cp -r /home ../workspace/scripts/` gave `Permission denied`**: Because you ran `cp` without `sudo`, Linux blocked you from reading inside `/home/ubuntu` and `/home/test`, and blocked copying protected hidden files (`.ssh`, `.bashrc`). Always use `sudo cp -r` and an absolute path (`/workspace/scripts/`).
2. **`rm: descend into write-protected directory 'linix'?`**: Happens when a folder is owned by `root` or has read-only permissions. Adding `-r` (recursive) and `-f` (force) with `sudo` (`sudo rm -rf linix`) destroys it immediately. *(⚠️ Be careful: `rm -rf` has no recycle bin!).*

---

## 🐳 9. Docker Container Quirks (Why `who` and `last` are blank)

1. **Why `who` and `last` don't show anything:** Normal Linux servers run background services that write login history to `/var/run/utmp` and `/var/log/wtmp`. Docker containers strip out these background logging services to stay fast and lightweight.
2. **How to check active processes instead:**
   ```bash
   # See every running program, command, and which user started it right now
   ps aux
   ```
3. **Ephemeral vs. Persistent Storage:** Anything saved inside `/home/devops` or `/home/himanshu` is wiped out if you run `docker-compose down`. Always store files you want to keep forever inside `/workspace/scripts` or `/workspace/data`.