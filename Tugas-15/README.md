## Bootcamp Digital Skill Sanbercode | DevOps - Batch 76

---

#### Tugas 15: (Automated Deployment) Integrasi Pipeline GitHub ke Production Server.

### 1️⃣ Add New Project Folder

```bash
$ mkdir Tugas-15
```

Now, we’re essentially moving from CI (test + build artifact) into CD (artifact delivery + remote execution).

---

### 2️⃣ Step-by-Step Setup

#### Step 1 — Update / Create Workflow File (Final CI/CD)

- ✅ Use AI to generate GitHub Actions workflow YAML

- **Prompt:**

> _I have already tested Workflow YAML for CI Core (Test-Pipeline) that create basic frame & protect MY_SECRET_KEY.
> Then I have already also tested Workflow YAML for Build Automation.
> Now, create a CI/CD YAML configuration for [Select: GitHub Actions / GitLab CI] that continues the previous build job. Create a new job named 'deploy-to-vps'. This job should download the 'hasil-build-web' artifact that was created. After that, use a secure SCP configuration to transfer the 'dist/' folder to the VPS. Then run an SSH command to log in to the VPS and execute the command 'echo Deployment Done!'. Make sure all credentials such as Host, Username, and SSH Private Key are called using the Secrets/Variables system.
> Guide me step-by-step to define it, setup it, and test it in CI/CD Pipeline._

Below is a clean continuation of your pipeline with a new job: `deploy-to-vps`.

- ✅ If you want to create a separate new workflow (`deploy.yml`), follow the steps below:

```bash
# Navigate to your project (pwd - main dir)
$ cd BDS-Sanbercode-Devops-Batch-76

# Create GitHub Actions directory
$ mkdir -p .github/workflows

# Create workflow file (in WSL/Linux): main.yml
$ nano .github/workflows/deploy.yml
```

#### Step 2 — Full YAML Configuration (CI + Build Artifact + Deploy)

✍️ Paste the YAML from AI above, then Save.

✅ **Option 1:** Combine new workflow (Deploy) with your previous workflow (CI + Build Artifact) like this:

```YAML
name: CI-CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  Test-Pipeline:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Run Basic Test
        run: echo "Running tests..."

  build-app:
    runs-on: ubuntu-latest
    needs: Test-Pipeline
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Create Build Artifact
        run: |
          mkdir -p dist
          echo "This is the result of an automated build artifact" > dist/index.html

      - name: Upload Artifact
        uses: actions/upload-artifact@v4
        with:
          name: hasil-build-web
          path: dist/

  deploy-to-vps:
    runs-on: ubuntu-latest
    needs: build-app

    steps:
      - name: Download Artifact
        uses: actions/download-artifact@v4
        with:
          name: hasil-build-web
          path: dist/

      - name: Setup SSH Key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519

      - name: Add VPS to Known Hosts
        run: |
          ssh-keyscan -H ${{ secrets.VPS_HOST }} >> ~/.ssh/known_hosts

      - name: Copy Files to VPS (SCP)
        run: |
          scp -r dist/* ${{ secrets.VPS_USER }}@${{ secrets.VPS_HOST }}:/var/www/html/

      - name: Execute Remote Command (SSH)
        run: |
          ssh ${{ secrets.VPS_USER }}@${{ secrets.VPS_HOST }} "echo Deployment Done!"
```

✅ **Option 2:** Create a separate YAML file (`deploy.yml`) using third-party actions (`appleboy/scp-action@v0.1.7`):

```YAML
name: CI/CD Pipeline Production

on:
  push:
    branches:
      - main  # refers to your repo-branch (main / master)

jobs:
  build-app:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Build Application
        run: |
          mkdir -p dist
          echo "This is the result of an automated build artifact" > dist/index.html

      - name: Upload Artifact
        uses: actions/upload-artifact@v4
        with:
          name: hasil-build-web
          path: dist/

  deploy-to-vps:
    runs-on: ubuntu-latest
    needs: build-app

    steps:
      # 1. Download artifact from previous job
      - name: Download Build Artifact
        uses: actions/download-artifact@v4
        with:
          name: hasil-build-web
          path: dist/

      # 2. Transfer files using SCP (via appleboy action)
      - name: Copy Artifact to VPS
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USERNAME }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          source: "dist/*"
          target: "/var/www/aplikasi-saya"
          strip_components: 1   # removes "dist/" nesting (important)

      # 3. Execute remote command via SSH
      - name: Execute Remote Command
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USERNAME }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            echo "Memulai proses deployment di VPS..."
            ls -la /var/www/aplikasi-saya
            echo "Deployment Production Sukses!"
```

- ✅ Preview - Create new workflow (`deploy.yml`)
  ![Bootcamp Digital Skill Sanbercode](ss-github-actions/xxx.png)

  <br>

#### Step 3 — Required Secrets Setup (CRITICAL)

- [x] Go to:
  - GitHub Repo → **Settings** → **Secrets and variables** → Actions → **New repository secret**.
- [x] Create:

| Secret Name       | Value Example                  |
| ----------------- | ------------------------------ |
| `VPS_HOST`        | `***.***.***.***`              |
| `VPS_USER`        | `***`                          |
| `SSH_PRIVATE_KEY` | (your **private key** content) |

<br>

##### 🔐 How to Get SSH_PRIVATE_KEY

- [x] On your local machine (WSL Terminal / Shell):

```bash
#$ cd ~
$ cat ~/.ssh/id_ed25519
# It is 'Private key', instead of public key
```

- [x] Copy entire output:

```text
-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----
```

- [x] Paste into **GitHub Secret**.
  - ⚠️ Do NOT use .pub (public key)

- ✅ Preview - xxx
  ![Bootcamp Digital Skill Sanbercode](ss-github-actions/xxx.png)

<br>

#### Step 4 — VPS Preparation (One-Time Setup)

- [x] Login to VPS:

```bash
$ ssh root@your_vps_ip
```

- [x] **A. Ensure directory exists**

```bash
$ mkdir -p /var/www/html
# This is where your index.html file exist (placed)
```

- [x] **B. Add your PUBLIC key**

```bash
# (from your local machine) - Get a copy:
$ cat ~/.ssh/id_ed25519.pub

# Paste your key - On VPS:
$ nano ~/.ssh/authorized_keys
# Save and exit → then, Set Permissions for the .ssh folder
```

- [x] **C. Set correct permissions**

```bash
# on Server - Set correct permissions
$ chmod 700 ~/.ssh
$ chmod 600 ~/.ssh/authorized_keys
```

<br>

#### Step 5 — Pipeline Execution Flow

```text
Push to main
   ↓
Test-Pipeline (CI)
   ↓
build-app → creates dist/ + artifact
   ↓
deploy-to-vps
   ↓
Download artifact
   ↓
SCP → VPS:/var/www/html
   ↓
SSH → echo Deployment Done!
```

---

### 3️⃣ How to Test the Pipeline

#### Step 1 — Commit & Push

```bash
$ git add .
$ git commit -m "Test CI/CD deploy"
$ git push origin main
```

<br>

#### Step 2 — Monitor Actions

- [x] Go to:
  - GitHub → Actions → CI-CD Pipeline.
- [x] Create:
  - ✅ Test-Pipeline
  - ✅ build-app
  - ✅ deploy-to-vps

<br>

#### Step 3 — Verify Deployment

- [x] Open browser: `http://YOUR_VPS_IP`

- [x] Expected output:

```text
This is the result of an automated build artifact
```

---

### 4️⃣ Common Failure Points (Debug Fast)

❌ Permission denied (publickey)

→ Wrong SSH_PRIVATE_KEY or not added to VPS

❌ SCP fails

→ Check:

```bash
$ /var/www/html permissions
```

→ Fix:

```bash
$ chmod -R 755 /var/www/html
```

❌ Host verification failed

→ Ensure this step exists:

```YAML
$ ssh-keyscan -H ${{ secrets.VPS_HOST }} >> ~/.ssh/known_hosts
```

---

### 5️⃣ Security Best Practices

- Never hardcode IP, user, or keys
- Always use `secrets.*`
- Restrict VPS SSH:

```bash
$ PermitRootLogin prohibit-password
```

---

### 6️⃣ Optional Improvements (Next Level)

- Zero-downtime deploy (symlink release folders)
- Use rsync instead of scp
- Add rollback mechanism
- Integrate Nginx reload:

```bash
$ sudo systemctl reload nginx
```

<br>

- ✅ Preview - Verify artifact → Automation Build is success
  ![Bootcamp Digital Skill Sanbercode](ss-github-actions/xxx.png)

---
