pipeline {
    agent { label 'jenkins-agent' }

    tools {
        jdk 'java17'
        maven 'maven3'
    }

    environment {
	    APP_NAME = "register-app-pipeline"
            RELEASE = "1.0.0"
            DOCKER_USER = "ashfaque9x"
            DOCKER_PASS = 'dockerhub'
            IMAGE_NAME = "${DOCKER_USER}" + "/" + "${APP_NAME}"
            IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
	    JENKINS_API_TOKEN = credentials("JENKINS_API_TOKEN")
    }

    stages {

        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main',
                    credentialsId: 'github',
                    url: 'https://github.com/Swapna-09/register-app.git'
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
                withSonarQubeEnv(credentialsId: 'jenkins-sonarqube-token') {
                    sh "mvn sonar:sonar"
                }
            }
        }

        stage("Quality Gate") {
            steps {
                waitForQualityGate abortPipeline: false,
                                   credentialsId: 'jenkins-sonarqube-token'
            }
        }

        stage("Build & Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('', DOCKER_PASS) {
                        def docker_image = docker.build("${IMAGE_NAME}")
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push("latest")
                    }
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                sh '''
                docker run --rm \
                  -v /var/run/docker.sock:/var/run/docker.sock \
                  aquasec/trivy image \
                  2swapna/register-app-pipeline:latest \
                  --no-progress \
                  --scanners vuln \
                  --exit-code 0 \
                  --severity HIGH,CRITICAL \
                  --format table
                '''
            }
        }

        stage("Trigger CD Pipeline") {
            steps {
                sh """
                curl -v -k --user admin:${JENKINS_API_TOKEN} \
                -X POST \
                -H 'cache-control: no-cache' \
                -H 'content-type: application/x-www-form-urlencoded' \
                --data 'IMAGE_TAG=${IMAGE_TAG}' \
                'http://ec2-174-129-191-30.compute-1.amazonaws.com:8080/job/register-app-cd/buildWithParameters?token=gitops-token'
                """
            }
        }
    }

    post {
        failure {
            emailext(
                body: '''${SCRIPT, template="groovy-html.template"}''',
                subject: "${env.JOB_NAME} - Build #${env.BUILD_NUMBER} - Failed",
                mimeType: 'text/html',
                to: "2swapna69@gmail.com"
            )
        }

        success {
            emailext(
                body: '''${SCRIPT, template="groovy-html.template"}''',
                subject: "${env.JOB_NAME} - Build #${env.BUILD_NUMBER} - Successful",
                mimeType: 'text/html',
                to: "2swapna69@gmail.com"
            )
        }
    }
}
