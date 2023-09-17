pipeline {
    agent { label 'jenkins-Agent' }

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        APP_NAME = "java"
        RELEASE = "1.0.0"
        DOCKER_USER = "shan123456"
        DOCKER_PASS = 'dockerhub'
        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        SONAR_URL = "http://54.252.243.214:9000/"
        // Define JENKINS_API_TOKEN here if needed
    }

    stages {
        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/shantanudatarkar/register-app.git'
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
                    withSonarQubeEnv(credentialsId: 'sonarqube_tokens') {
                        sh "mvn sonar:sonar -Dsonar.host.url=${SONAR_URL} -Dsonar.login=admin -Dsonar.password=shantanu"
                    }
                }
            }
        }

        // Uncomment this section if you want to wait for a Quality Gate
        /*
        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonarqube_tokens'
                }
            }
        }
        */

        stage("Build & Push Docker Image") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub_id') {
                        docker_image = docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                        docker_image.push()
                        docker_image.push('latest')
                    }
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                script {
                    sh 'docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image shan123456/java:latest --no-progress --scanners vuln --exit-code 0 --severity HIGH,CRITICAL --format table'
                }
            }
        }

        stage('Cleanup Artifacts') {
            steps {
                script {
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker rmi ${IMAGE_NAME}:latest"
                }
            }
        }

        stage("Clone Manifest Repository") {
            steps {
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/shantanudatarkar/gitops-register-app-manifest.git'
            }
        }

        stage("Update Image Tag in Manifest") {
            steps {
                script {
                    def newImageTag = "${RELEASE}-${BUILD_NUMBER}"
                    sh "sed -i 's|image: shan123456/java:.*|image: shan123456/java:${newImageTag}|' /home/ubuntu/workspace/register-app/deployment.yaml"
                }
            }
        }

        stage("Commit and Push Changes") {
            steps {
                dir('/home/ubuntu/workspace/register-app/') {
                    sh 'git add .'
                    sh 'git commit -m "Update image tag"'
                    sh 'git push origin main'
                }
            }
        }

        stage("Deploy Application") {
            steps {
                dir('/home/ubuntu/workspace/register-app/') {
                    sh 'kubectl apply -f deployment.yaml'
                    sh 'kubectl apply -f service.yaml'
                }
            }
        }
    }
}
