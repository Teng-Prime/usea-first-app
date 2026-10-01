pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = '781493733610'
        AWS_REGION     = 'ap-southeast-2'
        ECR_REPO_NAME  = 'usea-first-appb'
        ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        SWARM_MANAGER  = '<54.252.212.173>'
        IMAGE_TAG      = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker Image version: ${IMAGE_TAG}"
                    sh "docker build -t ${ECR_REGISTRY}/${ECR_REPO_NAME}:${IMAGE_TAG} ."
                }
            }
        }

        stage('Push Image to ECR') {
            steps {
                script {
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                    sh "docker push ${ECR_REGISTRY}/${ECR_REPO_NAME}:${IMAGE_TAG}"
                }
            }
        }

        stage('Deploy to Swarm') {
            steps {
                script {
                    echo "Deploying update to Docker Swarm Cluster..."
                    sshagent(['ec2-ssh-key']) {
                        sh """
                            ssh -o StrictHostKeyChecking=no ubuntu@${SWARM_MANAGER} "
                                aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY} &&
                                ECR_REGISTRY=${ECR_REGISTRY} ECR_REPO_NAME=${ECR_REPO_NAME} IMAGE_TAG=${IMAGE_TAG} docker stack deploy --with-registry-auth -c docker-stack.yml mystack
                            "
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            sh "docker rmi ${ECR_REGISTRY}/${ECR_REPO_NAME}:${IMAGE_TAG} || true"
        }
    }
}