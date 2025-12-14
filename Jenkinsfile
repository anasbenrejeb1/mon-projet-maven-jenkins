pipeline {
    agent any  // Run on any available agent (e.g., built-in node)

    tools {
        maven 'Maven'  // Assumes you configured a Maven tool named 'Maven' in Manage Jenkins > Tools
        jdk 'JDK17'    // Assumes JDK 17 tool named 'JDK17'
    }

    environment {
        DOCKER_IMAGE_NAME = 'anasbenrejeb1/mon-projet-maven-jenkins'
        SONAR_PROJECT_KEY = 'tn.esprit:student-management'  // Matches your pom.xml artifactId:groupId
        SONAR_HOST_URL = 'http://localhost:9000'  // Adjust if SonarQube is elsewhere
    }

    stages {
        stage('Checkout') {
            steps {
                // Pulls code from your Git repo (configured in job or here)
                checkout scm  // Uses the repo URL/branch from your Jenkins job config
            }
        }

        stage('Build Maven') {
            steps {
                script {
                    // Clean, compile, test (skipped), package the JAR
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    // Assumes SonarQube Scanner plugin installed and server configured (name: 'Sonar')
                    def scannerHome = tool 'SonarScanner'  // Configure in Manage Jenkins > Tools
                    withSonarQubeEnv('Sonar') {  // 'Sonar' is the server name from Configure System
                        withEnv(["PATH+SONAR=${scannerHome}/bin"]) {
                            withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                                sh """
                                    mvn sonar:sonar \
                                    -Dsonar.projectKey=\${SONAR_PROJECT_KEY} \
                                    -Dsonar.host.url=\${SONAR_HOST_URL} \
                                    -Dsonar.login=\${SONAR_TOKEN}
                                """
                            }
                        }
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    // Wait for SonarQube analysis to finish and check gate status
                    timeout(time: 2, unit: 'MINUTES') {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Quality Gate failed: ${qg.status}. Check SonarQube dashboard for issues."
                        }
                        echo "Quality Gate passed! "
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Build the image using the Dockerfile (add it to repo root as per previous steps)
                    def image = docker.build("${DOCKER_IMAGE_NAME}:${BUILD_NUMBER}")
                    echo "Docker image built: ${image.id}"
                }
            }
        }

        stage('Push Docker Image to DockerHub') {
            steps {
                script {
                    // Assumes Docker Hub credentials added (ID: 'docker-hub-cred') in Manage Credentials
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-hub-cred') {
                        def image = docker.image("${DOCKER_IMAGE_NAME}:${BUILD_NUMBER}")
                        image.push()
                        image.push('latest')  // Optional: also tag as 'latest'
                    }
                    echo "Image pushed to Docker Hub!"
                }
            }
        }
    }

    post {
        always {
            // Cleanup regardless of success/failure
            sh 'mvn clean'
            // Optional: Archive artifacts
            archiveArtifacts artifacts: 'target/*.jar', allowEmptyArchive: true
        }
        success {
            echo 'Pipeline completed successfully! '
        }
        failure {
            echo 'Pipeline failed. Check logs above. '
        }
    }
}
