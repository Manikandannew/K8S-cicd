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
            steps {
                script {
                    def scannerHome = tool 'sonarscanner4'
                    withSonarQubeEnv('sonar-pro') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=rocket-nodejs \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=http://<your-sonarqube-server>:9000 \
                            -Dsonar.login=<your-sonar-token>
                        """
                    }
                }
            }
        }
    }
}
