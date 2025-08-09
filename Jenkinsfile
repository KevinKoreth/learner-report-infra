pipeline {
    agent any

    environment {
        AWS_REGION = "ap-south-1" // change to your region
        ECR_REPO_FE = "123456789012.dkr.ecr.ap-south-1.amazonaws.com/learner-report-cs-fe"
        ECR_REPO_BE = "123456789012.dkr.ecr.ap-south-1.amazonaws.com/learner-report-cs-be"
        K8S_NAMESPACE = "learner-report"
        HELM_RELEASE = "learner-report"
        CHART_PATH = "learner-report-chart"
        AWS_CREDENTIALS_ID = "aws-creds"           // AWS access key + secret in Jenkins credentials
        KUBECONFIG_CREDENTIALS_ID = "kubeconfig"   // Jenkins file credential for kubeconfig
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Login to ECR') {
            steps {
                withAWS(region: "${AWS_REGION}", credentials: "${AWS_CREDENTIALS_ID}") {
                    script {
                        sh '''
                            aws ecr get-login-password --region $AWS_REGION \
                                | docker login --username AWS --password-stdin $(echo $ECR_REPO_FE | cut -d'/' -f1)
                        '''
                    }
                }
            }
        }

        stage('Build & Push Docker Images') {
            parallel {
                stage('Frontend') {
                    steps {
                        script {
                            docker.build("${ECR_REPO_FE}:latest", "frontend/").push("latest")
                        }
                    }
                }
                stage('Backend') {
                    steps {
                        script {
                            docker.build("${ECR_REPO_BE}:latest", "backend/").push("latest")
                        }
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: KUBECONFIG_CREDENTIALS_ID, variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                        export KUBECONFIG=$KUBECONFIG_FILE
                        helm upgrade --install $HELM_RELEASE $CHART_PATH \
                            --namespace $K8S_NAMESPACE \
                            --create-namespace \
                            --set frontend.image=$ECR_REPO_FE:latest \
                            --set backend.image=$ECR_REPO_BE:latest
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ Deployment to $K8S_NAMESPACE successful!"
        }
        failure {
            echo "❌ Deployment failed!"
        }
    }
}
