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

        stage('Quality Gate') {
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
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
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo "✅ Pipeline SUCCESS!"
        }
        failure {
            echo "❌ Pipeline FAILED!"
        }
    }
}
