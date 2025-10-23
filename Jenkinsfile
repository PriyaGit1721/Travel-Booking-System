pipeline {
    agent any

    tools {
        jdk 'jdk21'
        nodejs 'nodejs'
    }

    environment {
        // SonarQube configuration and credentials
        SONARQUBE = 'sonar-server'
        AWS_CREDENTIALS = 'aws-creds'
        DOCKER_CREDENTIALS = 'docker-creds'

        // App and repo details
        IMAGE_NAME = 'travel-system'
        AWS_REGION = 'us-east-1'
        GIT_REPO = 'https://github.com/PriyaGit1721/Travel-Booking-System.git'
        GIT_BRANCH = 'master'
        SONAR_PROJECT_KEY = 'priya-project'
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo 'Cloning repository...'
                git branch: "${GIT_BRANCH}", url: "${GIT_REPO}"
            }
        }

        stage('Install Dependencies') {
            steps {
                echo 'Installing Node.js dependencies...'
                sh 'npm install'
            }
        }

        stage('SonarQube Code Analysis') {
            steps {
                script {
                    echo 'Running SonarQube scan...'
                    def scannerHome = tool 'sonarscanner'
                    withSonarQubeEnv("${SONARQUBE}") {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=$SONAR_HOST_URL \
                            -Dsonar.login=$SONAR_AUTH_TOKEN
                        """
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo 'Building Docker image...'
                    sh "docker build -t ${IMAGE_NAME}:latest ."
                }
            }
        }

        stage('Login to AWS ECR and Push Image') {
            steps {
                script {
                    withAWS(credentials: "${AWS_CREDENTIALS}", region: "${AWS_REGION}") {
                        script {
                            ECR_REPO_URI = sh(
                                script: "aws ecr describe-repositories --repository-names ${IMAGE_NAME} --query 'repositories[0].repositoryUri' --output text 2>/dev/null || aws ecr create-repository --repository-name ${IMAGE_NAME} --query 'repository.repositoryUri' --output text",
                                returnStdout: true
                            ).trim()

                            echo "ECR Repository URI: ${ECR_REPO_URI}"

                            sh """
                                aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO_URI}
                                docker tag ${IMAGE_NAME}:latest ${ECR_REPO_URI}:latest
                                docker push ${ECR_REPO_URI}:latest
                            """
                        }
                    }
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    echo 'Pushing image to Docker Hub...'
                    withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIALS}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh """
                            echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                            docker tag ${IMAGE_NAME}:latest $DOCKER_USER/${IMAGE_NAME}:latest
                            docker push $DOCKER_USER/${IMAGE_NAME}:latest
                        """
                    }
                }
            }
        }

        stage('Post SonarQube Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
    }

    post {
        success {
            echo "✅ Build, Scan, and Push to AWS ECR & Docker Hub completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed. Please check logs for details."
        }
    }
}
