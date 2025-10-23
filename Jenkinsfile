pipeline {
    agent any

    tools {
        jdk 'jdk21'
        nodejs 'nodejs'
    }

    environment {
        // Tool & credentials
        SONARQUBE = 'sonar-server'
        AWS_CREDENTIALS = 'aws-creds'
        DOCKER_CREDENTIALS = 'docker-creds'

        // Application configuration
        IMAGE_NAME = 'travel-booking-system'
        AWS_REGION = 'us-east-1'
        ECR_URL = '735263431599.dkr.ecr.us-east-1.amazonaws.com/travel-repo-ecr'

        // Git & SonarQube details
        GIT_REPO = 'https://github.com/PriyaGit1721/Travel-Booking-System.git'
        GIT_BRANCH = 'master'
        SONAR_PROJECT_KEY = 'priya-project'

        // Paths
        SONAR_SCANNER_PATH = '/opt/sonar-scanner/bin/sonar-scanner'
        DOCKER_COMPOSE_PATH = '/usr/bin/docker-compose'
    }

    stages {

        stage('Checkout Code') {
            steps {
                echo '📦 Cloning repository...'
                git branch: "${GIT_BRANCH}", url: "${GIT_REPO}"
            }
        }

        stage('Install Dependencies') {
            steps {
                echo '📥 Installing Node.js dependencies...'
                sh 'npm install'
            }
        }

        stage('SonarQube Code Analysis') {
            steps {
                echo '🔍 Running SonarQube scan...'
                withSonarQubeEnv("${SONARQUBE}") {
                    sh """
                        ${SONAR_SCANNER_PATH} \
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=$SONAR_HOST_URL \
                        -Dsonar.login=$SONAR_AUTH_TOKEN
                    """
                }
            }
        }

        stage('Build & Run with Docker Compose') {
            steps {
                echo '🐳 Building and running app with Docker Compose...'
                sh """
                    ${DOCKER_COMPOSE_PATH} down
                    ${DOCKER_COMPOSE_PATH} build
                    ${DOCKER_COMPOSE_PATH} up -d
                """
            }
        }

        stage('Push App Image to AWS ECR') {
            steps {
                script {
                    echo '🚀 Logging into AWS ECR and pushing image...'
                    withAWS(credentials: "${AWS_CREDENTIALS}", region: "${AWS_REGION}") {
                        sh """
                            docker build -t ${IMAGE_NAME}:latest .
                            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_URL}
                            docker tag ${IMAGE_NAME}:latest ${ECR_URL}:latest
                            docker push ${ECR_URL}:latest
                        """
                    }
                }
            }
        }

        stage('Push App Image to Docker Hub') {
            steps {
                echo '📤 Pushing image to Docker Hub...'
                withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIALS}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        docker build -t ${IMAGE_NAME}:latest .
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker tag ${IMAGE_NAME}:latest $DOCKER_USER/${IMAGE_NAME}:latest
                        docker push $DOCKER_USER/${IMAGE_NAME}:latest
                    """
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                echo '🧩 Checking SonarQube Quality Gate result...'
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

    }

    post {
        success {
            echo "✅ SUCCESS: Build, scan, and push completed successfully!"
        }
        failure {
            echo "❌ FAILURE: Please check pipeline logs for details."
        }
    }
}



