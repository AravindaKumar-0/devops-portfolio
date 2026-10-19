
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        IMAGE_NAME = 'devops-portfolio'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/AravindaKumar-0/devops-portfolio.git'
            }
        }

        stage('Validate Source') {
            steps {
                sh '''
                    set -eu
                    test -f index.html
                    test -f Dockerfile
                    test -f css/style.css
                    echo "Source files verified."
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu
                    docker build \
                        -t ${IMAGE_NAME}:${BUILD_NUMBER} \
                        -t ${IMAGE_NAME}:latest .
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu

                        LOCAL_IMAGE="${IMAGE_NAME}:${BUILD_NUMBER}"
                        REMOTE_IMAGE="${DH_USER}/${IMAGE_NAME}"

                        # Tag local images with the Docker Hub repository name.
                        docker tag "$LOCAL_IMAGE" "${REMOTE_IMAGE}:${BUILD_NUMBER}"
                        docker tag "$LOCAL_IMAGE" "${REMOTE_IMAGE}:latest"

                        # Authenticate using the Jenkins-managed access token.
                        echo "$DH_TOKEN" | docker login \
                            --username "$DH_USER" \
                            --password-stdin

                        # Publish both versioned and latest tags.
                        docker push "${REMOTE_IMAGE}:${BUILD_NUMBER}"
                        docker push "${REMOTE_IMAGE}:latest"

                        docker logout
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Portfolio image built and pushed to Docker Hub.'
        }

        failure {
            echo 'FAILURE: Inspect the failed stage in the Jenkins console.'
        }

        always {
            echo "Build result: ${currentBuild.currentResult}"
            echo "Build number: ${BUILD_NUMBER}"
        }
    }
}
