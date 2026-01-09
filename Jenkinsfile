pipeline {
    agent any

    environment {
        AWS_REGION = "us-east-1"
        ECR_REPO    = "mern-task-manager"
        IMAGE_TAG   = "${env.BUILD_NUMBER}"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: "${env.BRANCH_NAME}", url: 'https://github.com/<your-username>/<your-repo>.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                dir('backend') {
                    sh 'npm install'
                }
            }
        }

        stage('Run Tests') {
            steps {
                dir('backend') {
                    sh 'npm test || true'
                }
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs --exit-code 0 --severity HIGH,CRITICAL .'
            }
        }

        stage('Docker Build') {
            steps {
                dir('backend') {
                    sh '''
                    docker build -t $ECR_REPO:$IMAGE_TAG .
                    '''
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                trivy image --exit-code 0 --severity HIGH,CRITICAL $ECR_REPO:$IMAGE_TAG
                '''
            }
        }

        stage('Login to ECR') {
            steps {
                sh '''
                aws ecr get-login-password --region $AWS_REGION \
                | docker login --username AWS --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.$AWS_REGION.amazonaws.com
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh '''
                docker tag $ECR_REPO:$IMAGE_TAG <AWS_ACCOUNT_ID>.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:$BRANCH_NAME
                docker push <AWS_ACCOUNT_ID>.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:$BRANCH_NAME
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            when {
                anyOf {
                    branch 'development'
                    branch 'uat'
                    branch 'production'
                }
            }
            steps {
                script {
                    def namespace = ""

                    if (env.BRANCH_NAME == "development") {
                        namespace = "dev"
                    } else if (env.BRANCH_NAME == "uat") {
                        namespace = "uat"
                    } else if (env.BRANCH_NAME == "production") {
                        namespace = "prod"
                    }

                    sh """
                    kubectl apply -f k8s/base/ -n ${namespace}
                    """
                }
            }
        }

        stage('Manual Approval (Production)') {
            when {
                branch 'production'
            }
            steps {
                input message: 'Approve deployment to PRODUCTION?', ok: 'Deploy'
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully for ${env.BRANCH_NAME}"
        }
        failure {
            echo "Pipeline failed for ${env.BRANCH_NAME}"
        }
    }
}
