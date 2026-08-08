pipeline {
    agent any

    tools {
        maven 'maven'
    }

    environment {
        ImageName = 'spring-petclinic'
        BUILD_TAG = "latest"
    }

    stages {
        stage('Checkout From Git') {
            steps {
                git branch: 'main', url: 'https://github.com/shivakathi1995/spring-petclinic.git'
            }
        }

        stage('Maven Validate') {
            steps {
                echo 'Validating the project...'
                sh 'mvn validate'
            }
        }

        stage('Maven Compile') {
            steps {
                echo 'Compiling the project...'
                sh 'mvn compile'
            }
        }

        stage('Maven Test') {
            steps {
                echo 'Running tests...'
                sh 'mvn test'
            }
        }

        stage('Maven Package') {
            steps {
                echo 'Packaging the project...'
                sh 'mvn package'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                sh '''
                    docker build -t ${ImageName}:${BUILD_TAG} .
                '''
            }
        }

        stage('Trivy Scan') {
            steps {
                echo 'Running Trivy scan...'
                sh '''
                    trivy image --format table --severity HIGH,CRITICAL \
                        --output trivy-report.txt ${ImageName}:${BUILD_TAG}
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.txt'
                }
            }
        }

        stage('Login to ACR and Push Image') {
            steps {
                withCredentials([
                    usernamePassword(credentialsId: 'AZURE_CREDENTIALS', usernameVariable: 'AZURE_CLIENT_ID', passwordVariable: 'AZURE_CLIENT_SECRET')
                ]) {
                    script {
                        echo "Logging into Azure Container Registry..."
                        sh '''
                            az acr login --name shivapetclinicacr
                            docker tag ${ImageName}:${BUILD_TAG} shivapetclinicacr.azurecr.io/${ImageName}:${BUILD_TAG}
                            docker push shivapetclinicacr.azurecr.io/${ImageName}:${BUILD_TAG}
                        '''
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo 'Deploying application to AKS cluster...'
                    sh '''
                        az aks get-credentials --resource-group RGJENKINS07AUGUST --name petclinic-aks --overwrite-existing
                        kubectl apply -f petclinic.yaml
                        kubectl get all
                    '''
                }
            }
        }
    }
}