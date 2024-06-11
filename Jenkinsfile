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
                    GIT_TAG = sh(returnStdout: true, script: "git describe --tags --match 'v[0-9]*' --abbrev=0 || echo ''").trim()
                    if (GIT_TAG == '') {
                        GIT_TAG = 'latest'
                    }
                    echo "Git Tag: ${GIT_TAG}"
                }
            }
        }
        stage('Parse Config') {
            steps {
                script {
                    // Parse the configuration file
                    def config = readYaml file: 'build-config.yaml'
                    // Set the configuration variables App
                    APP_NAME = config.config.app.name
                    APP_DESCRIPTION = config.config.app.description
                    MAINTAINER = config.config.maintainer
                    // Set the configuration variables Docker
                    DOCKER_REGISTRY = config.config.registry.url
                    DOCKER_IMAGE = config.config.image.name
                    DOCKER_USERNAME = config.config.registry.username
                    DOCKER_URL = $DOCKER_REGISTRY + '/' + $DOCKER_USERNAME + '/' + $DOCKER_IMAGE
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
                            def trivyOutput = sh(script: "trivy fs --scanners vuln,secret,misconfig --exit-code 1 --severity CRITICAL .", returnStdout: true).trim()

                            // Display Trivy scan results
                            println trivyOutput

                            // Check if vulnerabilities were found
                            if (trivyOutput.contains("Total: 0")) {
                                echo "No vulnerabilities found in the source code."
                            } else {
                                echo "Vulnerabilities found in the source code."
                            }
                        }
                    }
                }
                stage('Scanning Source Code with SonarQube') {
                    steps {
                        script {
                            // Run SonarQube analysis
                            withSonarQubeEnv('SonarQube') {
                                sh 'sonar-scanner'
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
                    sh "docker build -t ${DOCKER_URL}:${GIT_TAG} ."
                }
            }
        }
        stage('Scan Docker Image') {
            steps {
                script {
                    // Run Trivy to scan the Docker image
                    def trivyOutput = sh(script: "trivy image --exit-code 1 --severity CRITICAL --light ${DOCKER_URL}:${GIT_TAG}", returnStdout: true).trim()

                    // Display Trivy scan results
                    println trivyOutput

                    // Check if vulnerabilities were found
                    if (trivyOutput.contains("Total: 0")) {
                        echo "No vulnerabilities found in the Docker image."
                    } else {
                        echo "Vulnerabilities found in the Docker image."
                    }
                }
            }
        }
        stage('Docker Login') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'github', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD')]) {
                        echo 'Logging in to Docker registry...'
                        sh 'echo $DOCKER_PASSWORD | docker login $DOCKER_REGISTRY -u $DOCKER_USER --password-stdin'
                    }
                }
            }
        }
        stage('Image Push') {
            steps {
                script {
                    echo 'Pushing Docker image...'
                    sh "docker push ${DOCKER_URL}:${GIT_TAG}"
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
