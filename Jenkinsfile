pipeline {
    agent any

    environment {
        AWS_REGION        = 'us-east-1'
        ECR_REPO          = '564882306271.dkr.ecr.us-east-1.amazonaws.com/manikandan_repo'
        IMAGE_TAG         = "v${BUILD_NUMBER}"
        EKS_CLUSTER_NAME  = 'vgs_cluster'
        HELM_CHART_PATH   = './Helm'
        HELM_RELEASE_NAME = 'myrocket'
        WORKSPACE_PATH    = '/var/lib/jenkins/workspace/cicdmani'
    }

    stages {

        // STAGE 1: Pull code from GitHub
        stage('SCM Checkout') {
            steps {
                deleteDir()
                git branch: 'main',
                    credentialsId: 'git-cred',
                    url: 'https://github.com/Manikandannew/K8S-cicd.git'
                sh 'ls -lart'
            }
        }

        // STAGE 2: SonarQube Code Quality Scan
        stage('SonarQube SAST') {
            steps {
                script {
                    def scannerHome = tool 'sonarscanner4'
                    withSonarQubeEnv('sonar-pro') {
                        sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=rocket-nodejs"
                    }
                }
            }
        }

        // STAGE 3: Install Node.js dependencies
        stage('Build') {
            steps {
                sh 'npm install'
            }
        }

        // STAGE 4: Build Docker Image
        stage('Docker Build') {
            steps {
                sh """
                    echo "Building Docker image: my_pvt_repo:${IMAGE_TAG}"
                    docker build -t my_pvt_repo:${IMAGE_TAG} .
                    docker images | grep my_pvt_repo
                """
            }
        }

        // STAGE 5: Push Docker Image to AWS ECR
        stage('Docker Push to ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-crendentails-aj'
                ]]) {
                    sh """
                        echo "Logging into ECR..."
                        aws ecr get-login-password --region ${AWS_REGION} | \
                            docker login --username AWS --password-stdin ${ECR_REPO}

                        echo "Tagging image..."
                        docker tag my_pvt_repo:${IMAGE_TAG} ${ECR_REPO}:${IMAGE_TAG}
                        docker tag my_pvt_repo:${IMAGE_TAG} ${ECR_REPO}:latest

                        echo "Pushing to ECR..."
                        docker push ${ECR_REPO}:${IMAGE_TAG}
                        docker push ${ECR_REPO}:latest

                        echo "Done: ${ECR_REPO}:${IMAGE_TAG}"
                    """
                }
            }
        }

        // STAGE 6: Deploy to EKS using Helm
        stage('Deploy to EKS') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-crendentails-aj'
                ]]) {
                    sh """
                        echo "Updating kubeconfig..."
                        aws eks update-kubeconfig \
                            --name ${EKS_CLUSTER_NAME} \
                            --region ${AWS_REGION}

                        echo "Creating Docker registry secret..."
                        kubectl create secret generic helm-secret \
                            --from-file=.dockerconfigjson=\$HOME/.docker/config.json \
                            --type=kubernetes.io/dockerconfigjson \
                            --dry-run=client -o yaml | kubectl apply -f -

                        echo "Packaging Helm chart..."
                        helm package ${HELM_CHART_PATH}

                        echo "Deploying with Helm..."
                        helm upgrade --install ${HELM_RELEASE_NAME} \
                            ${WORKSPACE_PATH}/myrocketapp-0.1.0.tgz \
                            --set image.repository=${ECR_REPO} \
                            --set image.tag=${IMAGE_TAG} \
                            --wait \
                            --timeout 5m

                        echo "=== Deployment Status ==="
                        helm ls
                        kubectl get pods -o wide
                        kubectl get svc
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Deployment Successful! Image: ${ECR_REPO}:${IMAGE_TAG}"
        }
        failure {
            echo '❌ Pipeline Failed! Check the logs above.'
        }
        always {
            sh "docker rmi my_pvt_repo:${IMAGE_TAG} || true"
            sh "docker rmi ${ECR_REPO}:${IMAGE_TAG} || true"
        }
    }
}
