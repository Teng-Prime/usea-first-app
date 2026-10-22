pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = '781493733610'          
        AWS_REGION     = 'ap-southeast-2'       
        ECR_REPO_NAME  = 'usea-first-appb'        
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
                    # ១. Authenticate Docker ជាមួយ Amazon ECR
                    aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                    
                    # ២. Build Docker Image ពី Dockerfile
                    docker build -t ${ECR_REPO_NAME}:latest .
                    
                    # ៣. Tag Image តាម ECR Repository URL Format
                    docker tag ${ECR_REPO_NAME}:latest ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:latest
                    
                    # ៤. Push Docker Image ចូល Amazon ECR
                    docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:latest
                    """
                }
            }
        }

        stage('3. Deploy to Docker Swarm') {
            steps {
                // ប្រើប្រាស់ SSH Private Key 'swarm-ssh-key' ពី Jenkins Credentials
                sshagent(['swarm-ssh-key']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${SWARM_MANAGER} "
                        # Authenticate ECR លើ Swarm Manager
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                        
                        # Update Swarm Service ឱ្យប្រើ Image ថ្មី
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