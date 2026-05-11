pipeline {
    agent any
    environment {
        AWS_REGION        = 'us-east-1'
        ECR_REPO          = '564882306271.dkr.ecr.us-east-1.amazonaws.com/manikandan_repo'
        IMAGE_TAG         = "v${BUILD_NUMBER}"
        EKS_CLUSTER_NAME  = 'vgs_cluster'
        KUBECONFIG_PATH   = '/opt/kube/config'
        HELM_CHART_PATH   = './Helm'
        HELM_RELEASE_NAME = 'myrocket'
    }
    stages {
        stage('SCM Checkout') {
            steps {
                deleteDir()
                git branch: 'main',
                    credentialsId: 'git-cred',
                    url: 'https://github.com/Manikandannew/K8S-cicd.git'
                sh 'ls -lart'
            }
        }

        stage('SonarQube SAST') {
            environment {
                SONAR_TOKEN = credentials('sonar-token')
            }
            steps {
                script {
                    def scannerHome = tool 'sonarscanner4'
                    withSonarQubeEnv('sonar-pro') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=rocket-nodejs \
                            -Dsonar.sources=. \
                            -Dsonar.token=${SONAR_TOKEN} \
                            -Dsonar.exclusions=**/node_modules/**,**/dist/**,**/build/** \
                            -Dsonar.javascript.node.maxspace=2048 \
                            -Dsonar.scanner.socketTimeout=300 \
                            -Dsonar.scanner.responseTimeout=300
                        """
                    }
                }
            }
        }

        stage('NPM Build') {
            steps {
                sh 'npm install'
            }
        }

        stage('Docker Build Image') {
            steps {
                script {
                    sh "docker build -t manikandan_repo:${IMAGE_TAG} ."
                    sh 'docker images'
                }
            }
        }

        stage('Docker Push to ECR') {
            steps {
                script {
                    withCredentials([[
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-crendentails-aj'
                    ]]) {
                        // Login to ECR
                        sh """
                            aws ecr get-login-password --region ${AWS_REGION} | \
                            docker login --username AWS \
                            --password-stdin ${ECR_REPO}
                        """

                        // Tag image for ECR
                        sh "docker tag manikandan_repo:${IMAGE_TAG} ${ECR_REPO}:${IMAGE_TAG}"

                        // Push to ECR
                        sh "docker push ${ECR_REPO}:${IMAGE_TAG}"

                        echo "✅ Image pushed: ${ECR_REPO}:${IMAGE_TAG}"
                    }
                }
            }
        }
    }

    post {
        always {
            // Clean up local Docker images
            sh "docker rmi manikandan_repo:${IMAGE_TAG} || true"
            sh "docker rmi ${ECR_REPO}:${IMAGE_TAG} || true"
            cleanWs()
        }
        success {
            echo "✅ Pipeline SUCCESS! Image: ${ECR_REPO}:${IMAGE_TAG}"
        }
        failure {
            echo "❌ Pipeline FAILED!"
        }
    }
}
