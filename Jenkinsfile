pipeline {
    agent any

    environment {
        AWS_REGION = "ap-south-1"
        ECR_REPO_FE = "123456789012.dkr.ecr.ap-south-1.amazonaws.com/learner-report-cs-fe"
        ECR_REPO_BE = "123456789012.dkr.ecr.ap-south-1.amazonaws.com/learner-report-cs-be"
        K8S_NAMESPACE = "learner-report"
        HELM_RELEASE = "learner-report"
        CHART_PATH = "learner-report-chart"
        AWS_CREDENTIALS_ID = "aws-creds"
        KUBECONFIG_CREDENTIALS_ID = "kubeconfig"
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
                    sh '''
                        aws ecr get-login-password --region $AWS_REGION \
                            | docker login --username AWS --password-stdin $(echo $ECR_REPO_FE | cut -d'/' -f1)
                    '''
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

        stage('Create/Update Kubernetes Secret') {
            steps {
                withCredentials([
                    string(credentialsId: 'ATLAS_URI', variable: 'ATLAS_URI'),
                    string(credentialsId: 'HASH_KEY', variable: 'HASH_KEY'),
                    string(credentialsId: 'JWT_SECRET_KEY', variable: 'JWT_SECRET_KEY')
                ]) {
                    withCredentials([file(credentialsId: KUBECONFIG_CREDENTIALS_ID, variable: 'KUBECONFIG_FILE')]) {
                        sh '''
                            export KUBECONFIG=$KUBECONFIG_FILE
                            kubectl create secret generic learner-report-cs-be-secrets \
                              --from-literal=ATLAS_URI="$ATLAS_URI" \
                              --from-literal=HASH_KEY="$HASH_KEY" \
                              --from-literal=JWT_SECRET_KEY="$JWT_SECRET_KEY" \
                              -n $K8S_NAMESPACE \
                              --dry-run=client -o yaml | kubectl apply -f -
                        '''
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
