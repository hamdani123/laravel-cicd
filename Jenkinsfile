pipeline {
    agent any
    stages {
        stage('Info') {
            steps {
                echo 'Starting the pipeline...'
            }
        }
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Get Git Tag') {
            steps {
                script {
                    GIT_TAG = sh(script: "git describe --tags --match 'v[0-9]*' --abbrev=0 || echo 'latest'", returnStdout: true).trim()
                    echo "Git Tag: ${GIT_TAG}"
                }
            }
        }
        // Parse the configuration file
        stage('Parse Config') {
            steps {
                script {
                    def configFile = 'build-config.yaml'
                    def yqCommand = { String query -> sh(script: "yq -C ${query} ${configFile}", returnStdout: true).trim() }

                    // Set the configuration variables App
                    APP_NAME = yqCommand(".config.app.name")
                    APP_DESCRIPTION = yqCommand(".config.app.description")
                    MAINTAINER = yqCommand(".config.maintainer")
                    // Set the configuration variables Docker
                    DOCKER_REGISTRY = yqCommand(".config.registry.url")
                    DOCKER_IMAGE = yqCommand(".config.registry.image")
                    DOCKER_USERNAME = yqCommand(".config.registry.username")
                    DOCKER_URL = "${DOCKER_REGISTRY}/${DOCKER_USERNAME}/${DOCKER_IMAGE}:${GIT_TAG}"
                    // Set the configuration SonarQube
                    SONAR_PROJECT_KEY = yqCommand(".config.sonarqube.project_key")

                    // Display the configuration
                    echo "App Name: ${APP_NAME}"
                    echo "App Description: ${APP_DESCRIPTION}"
                    echo "Maintainer: ${MAINTAINER}"
                    echo "Docker URL: ${DOCKER_URL}"
                    echo "Version: ${GIT_TAG}"
                }
            }
        }
        stage('Parallel Scanning') {
            parallel {
                stage('Scanning Source Code with Trivy') {
                    steps {
                        script {
                            // Run Trivy to scan the source code
                            def trivyOutput = sh(script: "trivy fs --scanners vuln,secret,misconfig .", returnStdout: true).trim()
                            // Display Trivy scan results
                            println trivyOutput
                            // Check if vulnerabilities were found
                            if (trivyOutput.contains("Failures: 0")) {
                                echo "No vulnerabilities found in the source code."
                            } else {
                                echo "Vulnerabilities found in the source code."
                                // Uncomment the following line to fail the build if vulnerabilities are found
                                // error "Vulnerabilities found in the source code."
                            }
                        }
                    }
                }
                stage('Scanning Source Code with SonarQube') {
                    steps {
                        script {
                            withSonarQubeEnv('SonarQube') {
                                echo 'Running SonarQube Scanner...'
                                sh "/opt/sonar-scanner/bin/sonar-scanner -Dsonar.projectKey=${SONAR_PROJECT_KEY} -Dsonar.sources=."
                            }
                        }
                    }
                }
            }
        }
        stage('Build') {
            steps {
                script {
                    echo 'Building Docker image...'
                    sh "docker build -t ${DOCKER_URL} ."
                }
            }
        }
        stage('Scan Docker Image') {
            steps {
                script {
                    // Run Trivy to scan the Docker image
                    def trivyOutput = sh(script: "trivy image --scanners vuln ${DOCKER_URL}", returnStdout: true).trim()
                    // Display Trivy scan results
                    println trivyOutput
                    // Check if vulnerabilities were found
                    if (trivyOutput.contains("Total: 0")) {
                        echo "No vulnerabilities found in the Docker image."
                    } else {
                        echo "Vulnerabilities found in the Docker image."
                        // Uncomment the following line to fail the build if vulnerabilities are found
                        // error "Vulnerabilities found in the Docker image."
                    }
                }
            }
        }
        stage('Docker Login') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'container_registry', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD')]) {
                        echo "Logging in to Docker registry ${DOCKER_REGISTRY}"
                        sh "echo $DOCKER_PASSWORD | docker login ${DOCKER_REGISTRY} -u $DOCKER_USER --password-stdin"
                    }
                }
            }
        }
        stage('Image Push') {
            steps {
                script {
                    echo 'Pushing Docker image...'
                    sh "docker push ${DOCKER_URL}"
                }
            }
        }
        stage('Deploy to Server') {
            steps {
                script {
                    withCredentials([
                        sshUserPrivateKey(credentialsId: 'server_deployment', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER'),
                        string(credentialsId: 'ip_server_deployment', variable: 'SERVER'),
                        usernamePassword(credentialsId: 'container_registry', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD')
                    ]) {
                        sshagent(['server_deployment']) {
                            sh "[ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh"
                            sh "ssh-keyscan -t rsa,dsa ${SERVER} >> ~/.ssh/known_hosts"
                            sh "sed -i 's|DOCKER_URL|${DOCKER_URL}|g' docker-compose.yml"
                            sh "scp -o StrictHostKeyChecking=no docker-compose.yml ${SSH_USER}@${SERVER}:/opt/deployment-manifests/docker-compose.yml"
                            sh "ssh -o StrictHostKeyChecking=no ${SSH_USER}@${SERVER} 'sudo docker login ${DOCKER_REGISTRY} -u ${DOCKER_USER} -p ${DOCKER_PASSWORD}'"
                            sh "ssh -o StrictHostKeyChecking=no ${SSH_USER}@${SERVER} 'sudo docker compose -f /opt/deployment-manifests/docker-compose.yml up -d'"
                        }
                    }
                }
            }
        }
    }
    post {
        always {
            echo "I will always say Hello again!"
        }
        success {
            echo "I will only say Hello if the pipeline is successful!"
        }
        failure {
            echo "I will only say Hello if the pipeline has failed!"
        }
    }
}
