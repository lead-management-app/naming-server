pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = "${env.AWS_ACCOUNT_ID}"
        AWS_REGION     = "${env.AWS_DEFAULT_REGION}"
        SERVICE_NAME   = "lms/naming-server"
        REGISTRY_URL   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        APP_SERVER_IP  = "13.49.41.115"
        APP_SERVER_USER = "varata-admin"
        COMPOSE_SERVICE = "naming-server"
    }

    stages {
        stage('Step 1: Checkout') {
            steps {
                echo "checking out code from github."
                checkout scm
            }
        }

        stage('Step 2: Build Artifact & Push to AWS ECR.') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'aws-ecr-creds',
                                     passwordVariable: 'AWS_SECRET_ACCESS_KEY',
                                     usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                        sh '''
                        TOKEN=$(aws ecr get-login-password --region ${AWS_REGION})

                        ./mvnw clean package com.google.cloud.tools:jib-maven-plugin:build \
                            -Dimage=${REGISTRY_URL}/${SERVICE_NAME}:latest \
                            -Djib.to.auth.username=AWS \
                            -Djib.to.auth.password=$TOKEN
                        '''
                    }
                }
            }
        }

        stage('Step 3: Deploy to App Server') {
            steps {
                sshagent(['ssh-varata-server']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ${APP_SERVER_USER}@${APP_SERVER_IP} << 'EOF'
                        set -e

                        echo "Logging into ECR on App Server..."
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${REGISTRY_URL}

                        echo "Navigating to project folder..."
                        cd /home/varata-admin/varata-lms

                        echo "Updating images..."
                        docker compose pull ${COMPOSE_SERVICE}

                        echo "Restarting container..."
                        docker compose up -d ${COMPOSE_SERVICE}

                        echo "Cleaning up..."
                        docker image prune -f

                        echo "Deployment Successful."
EOF
                    """
                }
            }
        }
    }

    post {
            success {
                echo "Cleaning up workspace after successful build..."
                cleanWs()
            }
            failure {
                echo "Build failed. Keeping workspace for debugging..."
            }
        }
}