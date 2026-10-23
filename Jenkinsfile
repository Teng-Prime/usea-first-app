pipeline {
    agent any

    // ១. កំណត់ Parameters សម្រាប់ធ្វើ Rollback ដោយដៃ
    parameters {
        choice(name: 'ACTION', choices: ['deploy', 'rollback'], description: 'Select ACTION to perform')
        string(name: 'VERSION', defaultValue: 'v1.0.0', description: 'Target version tag (e.g., v1.0.0)')
    }

    // ២. កំណត់ Environment Variables សម្រាប់ AWS, ECR និង Docker Swarm[cite: 4]
    environment {
        AWS_ACCOUNT_ID = '781493733610'
        AWS_REGION     = 'ap-southeast-2'
        ECR_REPO_NAME  = 'usea-first-appb'
        SWARM_MANAGER  = '172.31.47.60'
        SERVICE_NAME   = 'my_web_app'
    }

    stages {
        // ៣. Stage កំណត់ Version និង Action
        stage('1. Determine Version & Action') {
            steps {
                script {
                    // ប្រសិនបើ trigger ដោយស្វ័យប្រវត្តិពី Git Tag លើ GitHub[cite: 3]
                    if (env.TAG_NAME) {
                        env.RELEASE_TAG = env.TAG_NAME
                        env.CURRENT_ACTION = 'deploy'
                    } else {
                        // ប្រសិនបើចុច Run ដោយដៃលើ Jenkins UI (Manual Rollback/Deploy)
                        env.RELEASE_TAG = params.VERSION
                        env.CURRENT_ACTION = params.ACTION
                    }
                    echo "Executing ACTION: ${env.CURRENT_ACTION} for VERSION: ${env.RELEASE_TAG}"
                }
            }
        }

        // ៤. Stage Build & Push Image ទៅ ECR (ដំណើរការតែពេល ACTION == 'deploy' ប៉ុណ្ណោះ)[cite: 3, 4]
        stage('2. Build & Push Image to ECR') {
            when {
                expression { env.CURRENT_ACTION == 'deploy' }
            }
            steps {
                script {
                    sh """
                    # Authenticate ជាមួយ Amazon ECR
                    aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                    
                    # Build Docker Image ជាមួយ Tag ច្បាស់លាស់ (ឧទាហរណ៍៖ usea-first-appb:v1.0.0)[cite: 3, 4]
                    docker build -t ${ECR_REPO_NAME}:${RELEASE_TAG} .
                    
                    # Tag Image តាម ECR URL Format
                    docker tag ${ECR_REPO_NAME}:${RELEASE_TAG} ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${RELEASE_TAG}
                    
                    # Push Versioned Image ចូល Amazon ECR[cite: 3, 4]
                    docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${RELEASE_TAG}
                    """
                }
            }
        }

        // ៥. Stage Deploy / Rollback ទៅកាន់ Docker Swarm[cite: 3, 4]
        stage('3. Deploy or Rollback on Docker Swarm') {
            steps {
                sshagent(['swarm-ssh-key']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${SWARM_MANAGER} "
                        # Authenticate ECR លើ Swarm Manager
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                        
                        # Update Swarm Service ទៅកាន់ Image Version ដែលចង់បាន (Rolling Update)[cite: 3, 4]
                        docker service update --image ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${RELEASE_TAG} --with-registry-auth ${SERVICE_NAME}
                    "
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline ${env.CURRENT_ACTION} completed successfully for version ${env.RELEASE_TAG}!"
        }
        failure {
            echo "Pipeline execution failed!"
        }
    }
}