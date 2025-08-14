# AWS CI/CD Setup: EC2 + Jenkins + Docker + Deployment

This repository demonstrates a **full CI/CD pipeline** on AWS using **EC2, Jenkins, Docker, and S3**, along with monitoring via CloudWatch.  
The pipeline builds, pushes, and deploys a simple web application containerized with Docker.

---

## **1. AWS EC2 Setup**

**Goal:** Launch a Linux server to host Jenkins, Docker, and run deployments.

**Steps:**
1. Launch EC2 instance (Ubuntu 22.04, t2.micro, 1 public IP)
2. Configure Security Group Inbound Rules:
   - SSH 22/tcp (your IP)
   - HTTP 80/tcp (0.0.0.0/0)
   - Jenkins 8080/tcp (0.0.0.0/0 for demo)
3. Download key pair `.pem`
4. Connect:
```bash
chmod 400 mykey.pem
ssh -i mykey.pem ubuntu@<EC2_PUBLIC_IP>
sudo apt update && sudo apt -y upgrade
```
# 2. Install Jenkins (CI Server)

Goal: Setup Jenkins LTS on EC2.

Steps:
```bash
sudo apt -y install openjdk-17-jdk
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt -y install jenkins
sudo systemctl enable --now jenkins
```

Setup:

Open http://<EC2_PUBLIC_IP>:8080
Get admin password:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Install suggested plugins + Git, Pipeline, Credentials Binding, Docker, Docker Pipeline
Create an admin user

# 3. Install Docker

Goal: Enable Jenkins to build & run containers.
```bash
sudo apt -y install docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker jenkins
sudo usermod -aG docker ubuntu
sudo systemctl restart jenkins
```

Verify:
```bash
docker ps
sudo -u jenkins -H docker ps
```

# 4. Prepare GitHub Repository (App + Dockerfile)

Steps:
```bash
mkdir myapp && cd myapp
cat > index.html <<'EOF'
<h1>Hello from AWS CI/CD!</h1>
EOF

cat > Dockerfile <<'EOF'
FROM nginx:alpine
COPY . /usr/share/nginx/html
EOF

echo -e "Dockerfile\n*.log\n" > .dockerignore

git init
git add .
git commit -m "Initial app"
git branch -M main
git remote add origin https://github.com/<you>/myapp.git
git push -u origin main
```

(Optional) GitHub Webhook: In your repo → Settings → Webhooks → add 
http://<EC2_PUBLIC_IP>:8080/github-webhook/. (You can also use “Poll SCM” in Jenkins if you can’t open 8080 to GitHub.)

Verify: Repo exists and shows Dockerfile & index.html.

# 5) Configure Jenkins pipeline (build, push, deploy)

Goal: On every commit, Jenkins builds an image, pushes to Docker Hub, and deploys the container on EC2.

Prep:

  * Create a Docker Hub account and repo (e.g., youruser/myapp).

  * In Jenkins: Manage Jenkins → Credentials → System → Global
    Add Username with password for Docker Hub, ID: dockerhub-creds.

Create a Jenkins Pipeline job named myapp-pipeline, select “Pipeline script from SCM” (Git) or paste this Jenkinsfile into the repo:
```bash
pipeline {
  agent any
  environment {
    IMAGE = "yourdockerhubuser/myapp"
    TAG   = "latest"                // or use env.BUILD_NUMBER
  }
  stages {
    stage('Checkout') {
      steps { git branch: 'main', url: 'https://github.com/youruser/myapp.git' }
    }
    stage('Build Image') {
      steps { sh 'docker build -t $IMAGE:$TAG .' }
    }
    stage('Push Image') {
      steps {
        withCredentials([usernamePassword(
           credentialsId: 'dockerhub-creds',
           usernameVariable: 'DOCKER_USER',
           passwordVariable: 'DOCKER_PASS'
        )]) {
          sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
          sh 'docker push $IMAGE:$TAG'
        }
      }
    }
    stage('Deploy') {
      steps {
        // Stop old container if running, then run new one on port 80
        sh '''
          (docker rm -f myapp || true)
          docker run -d --name myapp -p 80:80 $IMAGE:$TAG
        '''
      }
    }
  }
  post {
    always {
      sh 'docker image prune -f || true'
      archiveArtifacts artifacts: '**/*', fingerprint: true, onlyIfSuccessful: false
    }
  }
}
```

Verify:

  * Jenkins console shows successful stages.

  * Visit http://<EC2_PUBLIC_IP> → you should see “Hello from AWS CI/CD!”.

  * Docker Hub shows the pushed image tag.

# 6) Store build artifacts in S3 (optional but great for interviews)

Goal: Keep build outputs (e.g., zipped site, build manifest) in S3 for traceability.

Best practice: Attach an IAM Role to the EC2 with a minimal S3 policy (instead of storing keys).

   * Create an IAM Role (EC2 use case) and attach policy allowing s3:PutObject to your bucket.

Install & configure AWS CLI:
```bash
sudo apt -y install awscli
aws sts get-caller-identity   # should show the EC2 role, not an IAM user
```

Add an “Artifacts” stage to the Jenkinsfile (optional):
```bash
stage('Package & Upload to S3') {
  steps {
    sh '''
      zip -r build_${BUILD_NUMBER}.zip .
      aws s3 cp build_${BUILD_NUMBER}.zip s3://your-bucket/builds/
    '''
  }
}
```

Verify:

  * aws s3 ls s3://your-bucket/builds/ shows your zip.

  * Jenkins “Artifacts” tab shows archived files.

Pitfalls: No IAM role; wrong bucket name; region mismatch.

# 7) Monitoring with CloudWatch (health & alerts)

Goal: Get alerts if the instance or app has trouble.

Fast wins (no agent):

* Create CloudWatch Alarms for EC2:

  * CPUUtilization > 80% for 5 minutes

  * StatusCheckFailed > 0

Create an SNS Topic and subscribe your email for alerts.

Deeper visibility (with CloudWatch Agent):
```bash
# Install agent
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
sudo dpkg -i amazon-cloudwatch-agent.deb

# Minimal config file
cat <<'JSON' | sudo tee /opt/aws/amazon-cloudwatch-agent/bin/config.json
{
  "metrics": {
    "append_dimensions": { "InstanceId": "${aws:InstanceId}" },
    "metrics_collected": {
      "cpu": { "measurement": ["usage_system","usage_user","usage_idle"], "resources": ["*"], "totalcpu": true },
      "disk": { "measurement": ["used_percent"], "resources": ["*"] },
      "mem":  { "measurement": ["mem_used_percent"] }
    }
  },
  "logs": {
    "logs_collected": {
      "files": { "collect_list": [
        { "file_path": "/var/log/jenkins/jenkins.log", "log_group_name": "jenkins-log", "log_stream_name": "{instance_id}" }
      ]}
    }
  }
}
JSON

# Start agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json -s
```

Create CloudWatch dashboards & alarms for the new metrics (CPU, memory, disk). Optionally forward Nginx/container logs too.

Verify:

 * CloudWatch → Metrics show CPU/mem/disk.

 * SNS email received when you force high CPU (e.g., yes > /dev/null & then kill).
