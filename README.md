# របាយការណ៍កិច្ចការផ្ទះទី ២៖ ការដំឡើង CI/CD Pipeline ដោយប្រើ Jenkins, Docker Swarm, Amazon ECR និង Cloudflare Tunnel

---

## ១. ប្លង់រចនាសម្ព័ន្ធប្រព័ន្ធ (System Architecture & Network Setup)

ប្រព័ន្ធនេះត្រូវបានរៀបចំឡើងនៅលើ AWS EC2 Region `ap-southeast-2` ដោយបែងចែកជា ៣ Virtual Machines (Instances)៖

### តារាងកំណត់រចនាសម្ព័ន្ធ EC2 Instances

| ឈ្មោះ Server | ប្រភេទ OS | Private IP | Public IP | ភារកិច្ចសំខាន់ (Role) |
| --- | --- | --- | --- | --- |
| **Jenkins-Server** | Ubuntu 24.04 LTS | `172.31.39.254` | `[Jenkins Public IP]` | Master Server សម្រាប់ CI/CD, Build Docker Image & Run Cloudflare Tunnel |
| **Swarm-Manager** | Ubuntu 24.04 LTS | `172.31.47.60` | `[Manager Public IP]` | Swarm Leader សម្រាប់គ្រប់គ្រង Cluster និង Deploy Service |
| **Swarm-Worker** | Ubuntu 24.04 LTS | `172.31.44.129` | `[Worker Public IP]` | Worker Node សម្រាប់រត់ Application Containers |

### ការកំណត់ AWS Security Group Inbound Rules

* **Port 22 (SSH):** អនុញ្ញាត `0.0.0.0/0` សម្រាប់ Remote Management លើ EC2 ទាំង ៣។
* **Port 8080 (HTTP):** អនុញ្ញាត `0.0.0.0/0` លើ Jenkins-Server សម្រាប់បើក Jenkins Web UI។
* **Port 2377 (TCP):** សម្រាប់ Cluster Management Communication លើ Docker Swarm Manager។
* **Port 7946 (TCP/UDP):** សម្រាប់ Node-to-Node Control Traffic ក្នុង Swarm Cluster។
* **Port 4789 (UDP):** សម្រាប់ Overlay Network Traffic រវាង Containers លើ Nodes ផ្សេងគ្នា។

---

## ២. ការដំឡើង និងរៀបចំប្រព័ន្ធ (System Installation & Configuration)

### ២.១ ការដំឡើង Docker និងបង្កើត Swarm Cluster

១. ដំឡើង Docker Engine និង AWS CLI លើម៉ាស៊ីន Swarm-Manager និង Swarm-Worker៖

```bash
sudo apt update -y && sudo apt install -y docker.io awscli
sudo systemctl enable --now docker
sudo usermod -aG docker ubuntu

```

២. ចាប់ផ្តើមបង្កើត Swarm Cluster លើ **Swarm-Manager**៖

```bash
docker swarm init --advertise-addr 172.31.47.60

```

៣. ភ្ជាប់ **Swarm-Worker** ចូលទៅកាន់ Cluster ដោយប្រើ Join Token ដែលទទួលបានពី Manager៖

```bash
docker swarm join --token <JOIN_TOKEN> 172.31.47.60:2377

```

៤. ផ្ទៀងផ្ទាត់ស្ថានភាព Nodes លើ **Swarm-Manager**៖

```bash
docker node ls

```

### ២.២ ការដំឡើង Jenkins ជាមួយ Java 21

១. បន្ថែម Official Jenkins Repository និងដំឡើង Java 21 OpenJDK លើ **Jenkins-Server**៖

```bash
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key | sudo gpg --dearmor -o /usr/share/keyrings/jenkins-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.gpg] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update -y && sudo apt install -y openjdk-21-jdk jenkins docker.io awscli

```

២. បង្កើត Systemd Override Configuration ត្រង់ `/etc/systemd/system/jenkins.service.d/override.conf` ដើម្បីកំណត់ Environment Java 21៖

```ini
[Service]
Environment="JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64"
Environment="JENKINS_JAVA_CMD=/usr/lib/jvm/java-21-openjdk-amd64/bin/java"

```

៣. បន្ថែមសិទ្ធិ `jenkins` user ចូល `docker` group និង Restart Service៖

```bash
sudo usermod -aG docker jenkins
sudo systemctl daemon-reload
sudo systemctl restart jenkins

```

### ២.៣ ការបង្កើត Cloudflare Tunnel សម្រាប់ GitHub Webhook

ដើម្បីបង្កើត Public Secure Tunnel ដោយមិនចាំបាច់បើក Public IP ផ្ទាល់សម្រាប់ Webhook, ឧបករណ៍ `cloudflared` ត្រូវបានដំឡើងលើ **Jenkins-Server**៖

```bash
curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared.deb
nohup cloudflared tunnel --url http://localhost:8080 > cloudflared.log 2>&1 &
cat cloudflared.log | grep -o 'https://.*\.trycloudflare\.com'

```


---

## ៣. ការកំណត់ Jenkins Credentials និង GitHub Webhook

### ៣.១ ការកំណត់ Credentials លើ Jenkins UI

Credentials ចំនួន ២ ត្រូវបានបង្កើតក្នុង Jenkins (Manage Jenkins $\rightarrow$ Credentials $\rightarrow$ Global)៖

1. **`aws-credentials`:** ប្រភេទ *Username with password* (Username = AWS Access Key ID, Password = AWS Secret Access Key) សម្រាប់អនុញ្ញាតឱ្យ Jenkins អាច Authenticate និង Push Docker Image ចូល Amazon ECR។
2. **`swarm-ssh-key`:** ប្រភេទ *SSH Username with private key* (Username = `ubuntu`, Private Key = Content នៃ File `.pem`) សម្រាប់អនុញ្ញាតឱ្យ Jenkins Remote SSH ចូលទៅកាន់ Swarm-Manager ដើម្បីធ្វើការ Deploy។


### ៣.២ ការភ្ជាប់ GitHub Webhook

នៅក្នុង GitHub Repository Settings $\rightarrow$ Webhooks $\rightarrow$ Add Webhook៖

* **Payload URL:** `https://<CLOUDFLARE_TUNNEL_URL>/github-webhook/`
* **Content Type:** `application/json`
* **Trigger:** Just the push event


---

## ៤. ស្គ្រីប Pipeline (`Jenkinsfile`)

ស្គ្រីប `Jenkinsfile` ត្រូវបានដាក់ក្នុង Root Directory នៃ Source Code ដើមដើម្បីដំណើរការ CI/CD Automation៖

```groovy
pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = '123456789012'
        AWS_REGION     = 'ap-southeast-2'
        ECR_REPO_NAME  = 'my-web-app'
        SWARM_MANAGER  = '172.31.47.60'
        SERVICE_NAME   = 'my_web_app'
    }

    stages {
        stage('1. Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('2. Build & Push to ECR') {
            steps {
                script {
                    sh """
                    aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                    docker build -t ${ECR_REPO_NAME}:latest .
                    docker tag ${ECR_REPO_NAME}:latest ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:latest
                    docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:latest
                    """
                }
            }
        }

        stage('3. Deploy to Docker Swarm') {
            steps {
                sshagent(['swarm-ssh-key']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${SWARM_MANAGER} "
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                        docker service update --image ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:latest --with-registry-auth ${SERVICE_NAME}
                    "
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'CI/CD Pipeline Executed Successfully!'
        }
        failure {
            echo 'Pipeline Execution Failed!'
        }
    }
}

```

---

## ៥. លទ្ធផលនៃការអនុវត្ត (Execution Results)

នៅពេលមាន Developer ធ្វើការ `git push` កូដថ្មីចូលទៅកាន់ GitHub Repository:
១. GitHub ផ្ញើ Event តាមរយៈ Cloudflare Tunnel មកកាន់ Jenkins Webhook endpoint។
២. Jenkins ដំណើរការ Job ស្វ័យប្រវត្តិដោយធ្វើការ Checkout Source Code, Build Docker Image, Tag និង Push ទៅកាន់ Amazon ECR។
៣. Jenkins ធ្វើការ Remote SSH ចូល Swarm-Manager និងបញ្ជាឱ្យ Docker Swarm ធ្វើការ Rolling Update Application Service ដោយទាញយក Image ថ្មីពី Amazon ECR ដោយជោគជ័យ។

---

ការអនុវត្តគម្រោង CI/CD នេះបានបង្ហាញថាក្រុមការងារអាចរៀបចំ Pipeline ស្វ័យប្រវត្តិជាមួយ Jenkins, Docker Swarm, Amazon ECR និង Cloudflare Tunnel ដែលជួយបង្កើនប្រសិទ្ធភាពនៃការអភិវឌ្ឍ និងការប្រគល់កម្មវិធី (Software Delivery) ប្រកបដោយសុវត្ថិភាព គ្មាន Downtime និងកាត់បន្ថយការងារដោយដៃយ៉ាងច្រើន។