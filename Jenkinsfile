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
                        credentialsId: 'aws-crendentails-vgs'
                    ]]) {
                        sh """
                            aws ecr get-login-password --region ${AWS_REGION} | \
                            docker login --username AWS \
                            --password-stdin ${ECR_REPO}
                        """
                        sh "docker tag manikandan_repo:${IMAGE_TAG} ${ECR_REPO}:${IMAGE_TAG}"
                        sh "docker push ${ECR_REPO}:${IMAGE_TAG}"
                        echo "✅ Image pushed: ${ECR_REPO}:${IMAGE_TAG}"
                    }
                }
            }
        }

        stage('Deploy on EKS') {
            steps {
                script {
                    withCredentials([[
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-crendentails-vgs'  // ← fixed credentials ID
                    ]]) {
                        // Set KUBECONFIG
                        env.KUBECONFIG = '/opt/kube/config'

                        // Update kubeconfig for EKS
                        sh """
                            aws eks update-kubeconfig \
                            --name ${EKS_CLUSTER_NAME} \
                            --region ${AWS_REGION} \
                            --kubeconfig ${KUBECONFIG_PATH}
                        """

                        // Create Docker registry secret
                        sh """
                            kubectl create secret generic helm \
                                --from-file=.dockerconfigjson=/opt/docker/config.json \
                                --type=kubernetes.io/dockerconfigjson \
                                --dry-run=client -o yaml > secret.yaml
                            kubectl apply -f secret.yaml \
                                --kubeconfig ${KUBECONFIG_PATH}
                        """

                        // Package Helm chart
                        sh "helm package ${HELM_CHART_PATH}"

                        // Deploy with Helm
                        sh """
                            helm upgrade --install ${HELM_RELEASE_NAME} \
                            /var/lib/jenkins/workspace/K8s_pipeline/myrocketapp-0.1.0.tgz \
                            --kubeconfig ${KUBECONFIG_PATH} \
                            --set image.repository=${ECR_REPO} \
                            --set image.tag=${IMAGE_TAG}
                        """

                        // Verify deployment
                        sh "helm ls --kubeconfig ${KUBECONFIG_PATH}"
                        sh "kubectl get pods -o wide --kubeconfig ${KUBECONFIG_PATH}"
                        sh "kubectl get svc --kubeconfig ${KUBECONFIG_PATH}"
                    }
                }
            }
        }
    }

    post {
        always {
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
