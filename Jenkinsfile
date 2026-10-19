
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        IMAGE_NAME = 'devops-portfolio'
        DOCKERHUB_USER = 'imaravinda'
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
                    test -d css
                    test -f css/style.css

                    echo "Required portfolio files are present."
                '''
            }
        }

        stage('Validate HTML') {
            steps {
                sh '''
                    set -eu

                    if command -v tidy >/dev/null 2>&1; then
                        tidy -q -e index.html || {
                            echo "HTML validation reported errors."
                            exit 1
                        }
                    else
                        echo "HTML Tidy is not installed; skipping optional HTML validation."
                    fi
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu

                    docker build \
                      --pull \
                      -t ${IMAGE_NAME}:${BUILD_NUMBER} \
                      -t ${IMAGE_NAME}:latest .
                '''
            }
        }

        stage('Verify Docker Image') {
            steps {
                sh '''
                    set -eu

                    docker image inspect \
                      ${IMAGE_NAME}:${BUILD_NUMBER} >/dev/null

                    echo "Docker image built and verified."
                '''
            }
        }

        stage('Test Container') {
            steps {
                sh '''
                    set -eu

                    CONTAINER_NAME="portfolio-test-${BUILD_NUMBER}"
                    TEST_PORT=$((18000 + BUILD_NUMBER % 10000))

                    cleanup() {
                        docker rm -f "$CONTAINER_NAME" >/dev/null 2>&1 || true
                    }

                    trap cleanup EXIT

                    docker run -d \
                      --name "$CONTAINER_NAME" \
                      -p "127.0.0.1:${TEST_PORT}:80" \
                      ${IMAGE_NAME}:${BUILD_NUMBER}

                    for attempt in $(seq 1 15); do
                        if curl -fsS \
                            "http://127.0.0.1:${TEST_PORT}/" \
                            -o /tmp/portfolio-test.html; then
                            break
                        fi
                        sleep 2
                    done

                    grep -qi '<html' /tmp/portfolio-test.html
                    grep -qi '</html>' /tmp/portfolio-test.html

                    echo "Container HTTP smoke test passed."
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu

                        echo "$DH_TOKEN" | docker login \
                          --username "$DH_USER" \
                          --password-stdin

                        trap 'docker logout >/dev/null 2>&1 || true' EXIT

                        docker tag \
                          ${IMAGE_NAME}:${BUILD_NUMBER} \
                          ${DH_USER}/${IMAGE_NAME}:${BUILD_NUMBER}

                        docker tag \
                          ${IMAGE_NAME}:${BUILD_NUMBER} \
                          ${DH_USER}/${IMAGE_NAME}:latest

                        docker push ${DH_USER}/${IMAGE_NAME}:${BUILD_NUMBER}
                        docker push ${DH_USER}/${IMAGE_NAME}:latest
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Portfolio pipeline succeeded: image built, tested, and pushed.'
        }

        failure {
            echo 'Portfolio pipeline failed. Inspect the failed stage logs.'
        }

        always {
            echo "Build result: ${currentBuild.currentResult}"
            echo "Build number: ${BUILD_NUMBER}"
        }
    }
}
