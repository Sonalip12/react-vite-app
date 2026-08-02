pipeline {
    agent any

    tools {
        nodejs 'node-22'
    }

    environment {
        AWS_REGION = 'ap-southeast-2'
        CLUSTER_NAME = 'react-jenkins-eks'
        ECR_REPO = 'react-vite-app'
        APP_NAME = 'react-vite-app'
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Build React Application') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Prepare Image Details') {
            steps {
                script {
                    env.AWS_ACCOUNT_ID = sh(
                        script: 'aws sts get-caller-identity --query Account --output text',
                        returnStdout: true
                    ).trim()

                    env.ECR_REGISTRY = "${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com"
                    env.IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
                    env.IMAGE_URI = "${env.ECR_REGISTRY}/${env.ECR_REPO}:${env.IMAGE_TAG}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $IMAGE_URI .'
            }
        }

        stage('Push Docker Image') {
            steps {
                sh '''
                    aws ecr get-login-password --region $AWS_REGION | \
                    docker login --username AWS --password-stdin $ECR_REGISTRY

                    docker push $IMAGE_URI
                '''
            }
        }

        stage('Deploy to Amazon EKS') {
            steps {
                sh '''
                    aws eks update-kubeconfig \
                      --region $AWS_REGION \
                      --name $CLUSTER_NAME

                    sed "s|__IMAGE__|$IMAGE_URI|g" k8s/deployment.yaml | kubectl apply -f -
                    kubectl apply -f k8s/service.yaml
                '''
            }
        }

        stage('Verify Deployment Status') {
            steps {
                sh '''
                    kubectl rollout status deployment/$APP_NAME --timeout=180s
                    kubectl get pods
                    kubectl get service react-vite-service
                '''
            }
        }
    }

    post {
        always {
            sh 'docker image prune -f || true'
        }
    }
}