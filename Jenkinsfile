pipeline {
    agent any

    environment {
        // Azure & Container Registry Configuration
        REGISTRY_NAME     = 'shivapetclinicacr'                   // Replace with your ACR name (e.g., myacr07aug)
        ACR_URL           = "${REGISTRY_NAME}.azurecr.io"
        IMAGE_NAME        = 'spring-petclinic'
        IMAGE_TAG         = "${BUILD_NUMBER}"
        
        // Credentials IDs set up in Jenkins
        AZURE_CREDENTIALS = 'AZURE_CREDENTIALS'               // ID of your Azure Service Principal credential
        NOTIFICATION_EMAIL = 'rachamreddydivya98@gmail.com'   // Target email for pipeline alerts
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('SonarQube') {
            sh './mvnw org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                -Dsonar.projectKey=spring-petclinic \
                -Dsonar.host.url=http://4.221.131.153:9000'
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
                // Scans the local docker image for HIGH/CRITICAL vulnerabilities
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
                        sh "az login --service-principal -u ${AZURE_CLIENT_ID} -p ${AZURE_CLIENT_SECRET} --tenant ${TENANT_ID}"
                        sh "az acr login --name ${REGISTRY_NAME}"
                        
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
                        // Connect kubectl to your AKS cluster
                        sh "az aks get-credentials --resource-group myResourceGroup --name myAKSCluster --overwrite-existing"
                        
                        // Apply deployment manifests and update image tag
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