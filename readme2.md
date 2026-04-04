# Two-Tier-Flask-App — Jenkins CI/CD + DevSecOps Guide

This repository contains a simple two-tier Flask application (Flask + MySQL) and a Jenkins-based CI/CD pipeline with DevSecOps scanning (Trivy). This README documents how to set up Jenkins on AWS, create pipelines (master & agents), integrate Trivy file-system scans, build/push Docker images, and deploy via Docker Compose.

**Quick workflow:** CODE (GitHub) → BUILD (Docker) → SCAN (Trivy) → TEST → PUSH (Docker Hub) → DEPLOY (server)

**Diagram:**

![Workflow](screenshot/workflow.png)

**What this README covers:**
- Jenkins master setup on AWS EC2
- Jenkins agent/node setup (Docker + Docker Compose)
- Declarative Jenkins pipeline example (with Shared library usage)
- Trivy file-system scan integration and sample output
- Docker build, push, and deploy steps
- DevSecOps notes, credentials, and troubleshooting (includes console output reference)

**Screenshots:** add images into the `screenshot/` directory. Recommended filenames used in this README:
- `screenshot/jenkins-unlock.png` — Jenkins unlock page
- `screenshot/jenkins-create-job.png` — New pipeline job
- `screenshot/trivy-results.png` — sample Trivy results
- `screenshot/workflow.png` — workflow diagram

---

**Prerequisites**
- An AWS account to launch an EC2 instance (Ubuntu 22.04+ recommended)
- A GitHub repository with this project
- Docker Hub account (for image push)

**Ports**
- SSH: `22`
- Jenkins: `8080`
- Flask: `5000`
- MySQL: `3306`

---

**1) Jenkins master (EC2) — quick setup**

1. Launch an Ubuntu EC2 instance, attach a key pair, and open security group inbound rules for `22`, `8080`, `5000`, and `3306` as needed.

2. SSH to the instance:

```bash
ssh -i /path/to/key.pem ubuntu@<EC2_PUBLIC_IP>
```

3. Update and install Java (OpenJDK 21 recommended in your notes):

```bash
sudo apt-get update
sudo apt-get install -y fontconfig openjdk-21-jre
java --version
```

4. Install Jenkins (official Debian repo) and start it:

```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install -y jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins
```

5. Open `http://<EC2_PUBLIC_IP>:8080` in your browser. Retrieve initial admin password and unlock Jenkins:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Add screenshot: `screenshot/jenkins-unlock.png` for the Unlock Jenkins page.

6. Install suggested plugins and create an admin user via the Jenkins UI.

---

**2) Jenkins agent (node) setup**

On the agent machine (can be the same EC2 or separate instance):

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git docker.io docker-compose-plugin
sudo usermod -aG docker jenkins || true
newgrp docker || true
sudo systemctl restart jenkins || true
```

Notes:
- Ensure the `jenkins` user exists on the agent and is added to the `docker` group so Jenkins can run Docker commands.
- Verify Docker and Docker Compose with `docker --version` and `docker compose version`.

---

**3) Trivy (DevSecOps) — install & use**

Install Trivy on the Jenkins node(s) or master where scans run:

```bash
# Linux (apt-based example)
sudo apt-get install -y wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install -y trivy

# Or install via tarball/binary per https://trivy.dev
```

Basic file-system scan (as used in pipeline):

```bash
trivy fs . -o results.json
# view summarized JSON or convert to HTML if desired
```

Example Trivy caveats seen in pipeline run (from console output):
- Info: vulnerability and secret scanning enabled
- Warning: pip site-packages directory not found (license detection skipped)

Add screenshot: `screenshot/trivy-results.png` for a sample Trivy result.

---

**4) Jenkins pipeline (example)**

Create a Pipeline job and use a `Jenkinsfile` in the repository. Below is a production-friendly Declarative pipeline that mirrors workflow and uses a Shared library called `Shared`:

```groovy
@Library("Shared") _
pipeline {
   agent { label 'dev' }
   stages {
      stage('CODE') {
         steps {
            script {
               // clones repo (your Shared lib can provide helpers)
               git branch: 'main', url: 'https://github.com/pratikk-devops/two-tier-flask-app.git'
            }
         }
      }

      stage('Trivy File System Scan') {
         steps {
            sh 'trivy fs . -o results.json'
            // optionally archive results or fail on high severity
         }
      }

      stage('BUILD') {
         steps {
            sh 'docker build -t two-tier-flask-app .'
         }
      }

      stage('TEST') {
         steps {
            echo 'Developer/Tester writes the tests'
            // add unit/integration test commands here
         }
      }

      stage('Push to Docker Hub') {
         steps {
            withCredentials([usernamePassword(credentialsId: 'dockerHubCreds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
               sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
               sh 'docker tag two-tier-flask-app $DOCKER_USER/two-tier-flask-app:latest'
               sh 'docker push $DOCKER_USER/two-tier-flask-app:latest'
            }
         }
      }

      stage('DEPLOY') {
         steps {
            sh 'docker compose up -d --build flask-app'
         }
      }
   }
}
```

Important pipeline notes:
- Use `withCredentials` to avoid exposing secrets; avoid Groovy string interpolation for secrets.
- Fail the pipeline on high-severity Trivy findings if required (add logic to parse `results.json`).

---

**5) Docker build, tag, and push (commands)**

```bash
# build
docker build -t two-tier-flask-app .

# tag for Docker Hub
docker tag two-tier-flask-app <dockerhub-username>/two-tier-flask-app:latest

# login and push (use Jenkins credentials in pipeline)
docker login -u <user> -p <pass>
docker push <user>/two-tier-flask-app:latest
```

In the console output for this repo, the pipeline successfully built and pushed the image, and then deployed it with `docker compose up -d --build` (see the console log for exact output and timings).

---

**6) Shared Libraries, Agents & Nodes**

- Shared libraries let teams centralize pipeline helper functions. Example your pipeline already loads `@Library("Shared")` — put commonly used functions (e.g., `docker_push`, `trivy_fs`) there.
- Use labeled agents (e.g., `label 'dev'`) to direct jobs to specific nodes with Docker installed.

Example Shared library function signature (Groovy):

```groovy
def docker_push(String credId, String imageName) {
  withCredentials([usernamePassword(credentialsId: credId, usernameVariable: 'U', passwordVariable: 'P')]) {
   sh 'echo $P | docker login -u $U --password-stdin'
   sh "docker tag ${imageName} $U/${imageName}:latest"
   sh "docker push $U/${imageName}:latest"
  }
}
```

---

**7) Security & Best Practices**

- Do not store secrets in pipeline code. Use Jenkins Credentials Store for Docker Hub, Git tokens, SSH keys.
- Fail the pipeline for high/critical Trivy findings and notify via email/Slack.
- Scan base images (use `trivy image <image>` in addition to `trivy fs`).
- Use image signing and enforce scanning in a registry pre-push hook if available.

---

**8) Troubleshooting & console output reference**

- For a successful run example including Trivy messages, Docker build/push logs, and deployment status, see the pipeline console output
- Common findings in console output :
  - `trivy fs . -o results.json` produced info/warn messages about secret scanning and pip site-packages.
  - Docker build used `python:3.9-slim` and successfully built and tagged `two-tier-flask-app:latest`.
  - Docker push to Docker Hub succeeded (digest reported) and `docker compose up -d --build` recreated containers.

---

**9) Where to add screenshots**

Place images under `screenshot/` and reference them inline in this README (example used above). Recommended images are mentioned near the top.

---

