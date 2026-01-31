pipeline {
    agent { label 'Jenkins-Agent' }
    tools {
        jdk 'Java17'
        maven 'Maven3'
    }
    environment {
        APP_NAME = "register-app-pipeline"
        RELEASE = "1.0.0"
        DOCKER_USER = "2swapna"
        DOCKER_PASS = 'dockerhub'
        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        TRIVY_CACHE_DIR = '/mnt/ebs100/trivy' // Use the EBS-mounted Trivy cache
    }
    stages {
        stage("Cleanup Workspace & Docker") {
            steps {
                script {
                    // Clean workspace
                    cleanWs()

                    // Remove old Docker containers
                    sh 'docker container prune -f'

                    // Remove dangling Docker images
                    sh 'docker image prune -af'

                    // Remove unused volumes
                    sh 'docker volume prune -f'

                    // Remove unused networks (optional)
                    sh 'docker network prune -f'
                }
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/Ashfaque-9x/register-app'
            }
        }

        stage("Build Application") {
            steps {
                sh "mvn clean package"
            }
        }

        stage("Test Application") {
            steps {
                sh "mvn test"
            }
        }

        stage("SonarQube Analysis") {
            steps {
                script {
                    withSonarQubeEnv(credentialsId: 'jenkins-sonarqube-token') { 
                        sh "mvn sonar:sonar"
                    }
                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'jenkins-sonarqube-token'
                }
            }
        }

        stage("Build & Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('', DOCKER_PASS) {
                        docker_image = docker.build "${IMAGE_NAME}"
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push('latest')
                    }
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                script {
                    sh """
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                    -v ${TRIVY_CACHE_DIR}:/root/.cache/ aquasec/trivy \
                    image ${IMAGE_NAME}:${IMAGE_TAG} \
                    --no-progress --scanners vuln --exit-code 0 \
                    --severity HIGH,CRITICAL --format table
                    """
                }
            }
        }
    }
}
