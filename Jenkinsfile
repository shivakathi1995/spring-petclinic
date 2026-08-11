pipeline {
    agent any

    environment {
        // Azure & Container Registry Configuration
        REGISTRY_NAME      = 'shivapetclinicacr'
        ACR_URL            = "${REGISTRY_NAME}.azurecr.io"
        IMAGE_NAME         = 'spring-petclinic'
        IMAGE_TAG          = "${BUILD_NUMBER}"
        
        // Azure Tenant ID (Required for az login)
        AZURE_TENANT_ID    = '596f271a-e744-4410-9203-1836891565e6' // Replace with your actual Azure Tenant ID
        
        // Credentials IDs configured in Jenkins
        AZURE_CREDENTIALS  = 'AZURE_CREDENTIALS'        // Username/Password credential for Azure Service Principal
        SONAR_CREDENTIAL_ID = 'SonarQubetoken'              // Secret Text credential containing SonarQube token
        NOTIFICATION_EMAIL = 'rachamreddydivya98@gmail.com'
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: "${SONAR_CREDENTIAL_ID}", variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv('SonarQube') {
                        sh """
                            ./mvnw org.sonarsource.scanner.maven:sonar-maven-plugin:3.9.1.2184:sonar \
                            -Dsonar.projectKey=spring-petclinic \
                            -Dsonar.host.url=http://4.221.131.153:9000 \
                            -Dsonar.login=\${SONAR_TOKEN}
                        """
                    }
                }
            }
        }

        stage('Build & Test') {
            steps {
                sh './mvnw clean package -DskipTests=false'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${ACR_URL}/${IMAGE_NAME}:${IMAGE_TAG} ."
                    sh "docker tag ${ACR_URL}/${IMAGE_NAME}:${IMAGE_TAG} ${ACR_URL}/${IMAGE_NAME}:latest"
                }
            }
        }

        stage('Trivy Image Vulnerability Scan') {
            steps {
                // Scans local image for HIGH and CRITICAL vulnerabilities
                sh "trivy image --severity HIGH,CRITICAL ${ACR_URL}/${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('Login & Push to ACR') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${AZURE_CREDENTIALS}",
                    usernameVariable: 'AZURE_CLIENT_ID',
                    passwordVariable: 'AZURE_CLIENT_SECRET'
                )]) {
                    script {
                        // Authenticate with Azure CLI and log into ACR
                        sh "az login --service-principal -u ${AZURE_CLIENT_ID} -p ${AZURE_CLIENT_SECRET} --tenant ${AZURE_TENANT_ID}"
                        sh "az acr login --name ${shivapetclinicacr}"
                        
                        // Push images to ACR
                        sh "docker push ${ACR_URL}/${IMAGE_NAME}:${IMAGE_TAG}"
                        sh "docker push ${ACR_URL}/${IMAGE_NAME}:latest"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes (AKS)') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${AZURE_CREDENTIALS}",
                    usernameVariable: 'AZURE_CLIENT_ID',
                    passwordVariable: 'AZURE_CLIENT_SECRET'
                )]) {
                    script {
                        // Connect kubectl to your AKS cluster (Update RG name & AKS name if different)
                        sh "az aks get-credentials --resource-group RGJENKINS07AUGUST --name petclinic-aks-cluster --overwrite-existing"
                        
                        // Update container image tag on AKS
                        sh "kubectl set image deployment/spring-petclinic spring-petclinic=${ACR_URL}/${IMAGE_NAME}:${IMAGE_TAG}"
                    }
                }
            }
        }
    }

    post {
        success {
            emailext (
                to: "${NOTIFICATION_EMAIL}",
                subject: "SUCCESS: Jenkins Pipeline - ${env.JOB_NAME} [Build #${env.BUILD_NUMBER}]",
                mimeType: 'text/html',
                body: """
                    <h2 style="color: green;">Build Successful!</h2>
                    <p><strong>Job:</strong> ${env.JOB_NAME}</p>
                    <p><strong>Build Number:</strong> #${env.BUILD_NUMBER}</p>
                    <p><strong>URL:</strong> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    <p>The updated container image <code>${ACR_URL}/${IMAGE_NAME}:${IMAGE_TAG}</code> has been deployed to AKS.</p>
                """
            )
        }
        failure {
            emailext (
                to: "${NOTIFICATION_EMAIL}",
                subject: "FAILED: Jenkins Pipeline - ${env.JOB_NAME} [Build #${env.BUILD_NUMBER}]",
                mimeType: 'text/html',
                body: """
                    <h2 style="color: red;">Build Failed!</h2>
                    <p><strong>Job:</strong> ${env.JOB_NAME}</p>
                    <p><strong>Build Number:</strong> #${env.BUILD_NUMBER}</p>
                    <p><strong>Console Output:</strong> <a href="${env.BUILD_URL}console">${env.BUILD_URL}console</a></p>
                    <p>Please inspect the build logs to troubleshoot the failure.</p>
                """
            )
        }
    }
}