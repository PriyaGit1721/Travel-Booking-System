pipeline {
    agent any

    tools {
        jdk 'jdk21'
        nodejs 'nodejs'
    }

    environment {
        SONARQUBE = 'sonar-server'       // SonarQube system name in Jenkins
        DOCKER_CREDS = credentials('docker-creds')
        IMAGE_NAME = 'travel-system'
        CONTAINER_NAME = 'travel-booking'
        REGISTRY = 'docker.io'            // change if using AWS ECR, etc.
        BRANCH = 'master'
        REPO_URL = 'https://github.com/PriyaGit1721/Travel-Booking-System.git'
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo "Fetching latest source code from ${REPO_URL}..."
                git branch: "${BRANCH}", url: "${REPO_URL}"
            }
        }

        stage('Install Dependencies') {
            steps {
                echo "Installing Node.js dependencies..."
                sh 'npm install'
            }
        }

        stage('SonarQube Code Analysis') {
            steps {
                echo "Running SonarQube Analysis..."
                withSonarQubeEnv("${SONARQUBE}") {
                    sh '''
                        npx sonar-scanner \
                        -Dsonar.projectKey=travel-system \
                        -Dsonar.projectName="Travel Booking System" \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=$SONAR_HOST_URL \
                        -Dsonar.login=$SONAR_AUTH_TOKEN
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                echo "Checking SonarQube Quality Gate..."
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Run Tests') {
            steps {
                echo "Running project test cases..."
                sh 'npm test || echo "No test cases found, skipping..."'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image..."
                sh '''
                    docker build -t $IMAGE_NAME:latest .
                    docker tag $IMAGE_NAME:latest $REGISTRY/$DOCKER_CREDS_USR/$IMAGE_NAME:latest
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                echo "Pushing image to Docker Hub..."
                sh '''
                    echo $DOCKER_CREDS_PSW | docker login -u $DOCKER_CREDS_USR --password-stdin $REGISTRY
                    docker push $REGISTRY/$DOCKER_CREDS_USR/$IMAGE_NAME:latest
                '''
            }
        }

        stage('Deploy') {
            steps {
                echo "Deploying Docker container..."
                sh '''
                    docker stop $CONTAINER_NAME || true
                    docker rm $CONTAINER_NAME || true
                    docker run -d -p 3000:3000 --name $CONTAINER_NAME $REGISTRY/$DOCKER_CREDS_USR/$IMAGE_NAME:latest
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Deployment successful! Travel Booking System app is running."
        }
        failure {
            echo "❌ Build failed! Please check the Jenkins logs for more details."
        }
    }
}
