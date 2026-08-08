pipeline {
    agent any

    tools {
        maven 'maven'
    }

    environment {
        ImageName = 'spring-petclinic'
        BUILD_TAG = 'latest'
        ACR_NAME  = 'shivapetclinicacr'
        TENANT_ID = '596f271a-e744-4410-9203-1836891565e6'
    }

    stages {
        stage('Checkout From Git') {
            steps {
                git branch: 'main', url: 'https://github.com/shivakathi1995/spring-petclinic.git'
            }
        }

        stage('Maven Validate') {
            steps {
                echo 'Validating project dependencies...'
                sh 'mvn validate'
            }
        }

        stage('Maven Compile') {
            steps {
                echo 'Compiling source code...'
                sh 'mvn compile'
            }
        }

        stage('Maven Test') {
            steps {
                echo 'Running unit tests...'
                sh 'mvn test'
            }
        }

        stage('Maven Package') {
            steps {
                echo 'Packaging application into executable JAR...'
                sh 'mvn package -DskipTests'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    echo 'Running SonarQube Cloud code analysis...'
                    withSonarQubeEnv('SonarQube') {
                        sh '''
                            mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                                -Dsonar.organization=shiva2302 \
                                -Dsonar.projectKey=shiva2302_spring-petclinic
                        '''
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building container image...'
                sh 'docker build -t ${ImageName}:${BUILD_TAG} .'
            }
        }

        stage('Trivy Image Vulnerability Scan') {
            steps {
                echo 'Scanning Docker image for HIGH and CRITICAL vulnerabilities...'
                sh '''
                    trivy image --severity HIGH,CRITICAL \
                        --format table \
                        --output trivy-report.txt \
                        ${ImageName}:${BUILD_TAG}
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.txt', allowEmptyArchive: true
                }
            }
        }

        stage('Login to ACR and Push Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'AZURE_CREDENTIALS',
                        usernameVariable: 'AZURE_CLIENT_ID',
                        passwordVariable: 'AZURE_CLIENT_SECRET'
                    )
                ]) {
                    script {
                        echo "Authenticating with Azure and pushing container image to ACR..."
                        sh '''
                            az login --service-principal -u "$AZURE_CLIENT_ID" -p "$AZURE_CLIENT_SECRET" --tenant "${TENANT_ID}"
                            az acr login --name ${ACR_NAME}
                            docker tag ${ImageName}:${BUILD_TAG} ${ACR_NAME}.azurecr.io/${ImageName}:${BUILD_TAG}
                            docker push ${ACR_NAME}.azurecr.io/${ImageName}:${BUILD_TAG}
                        '''
                    }
                }
            }
        }

        stage('Deploy to Kubernetes (AKS)') {
            steps {
                script {
                    echo 'Deploying application to Azure Kubernetes Service cluster...'
                    sh '''
                        az aks get-credentials --resource-group RGJENKINS07AUGUST --name petclinic-aks --overwrite-existing
                        kubectl apply -f petclinic.yaml
                        kubectl get all
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Please inspect the stage logs.'
        }
    }
}