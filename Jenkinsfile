
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

                    echo "Required portfolio files are present."
                '''
            }
        }

        stage('Validate HTML') {
            steps {
                sh '''
                    set -eu

                    if command -v tidy >/dev/null 2>&1; then
                        tidy -q -e index.html
                    else
                        echo "HTML Tidy unavailable; optional validation skipped."
                    fi
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu

                    docker build --pull \
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

                    echo "Docker image verified."
                '''
            }
        }

        stage('Test Container') {
            steps {
                sh '''
                    set -eu

                    CONTAINER_NAME="portfolio-test-${BUILD_NUMBER}"
                    TEST_PORT=$((18000 + BUILD_NUMBER % 10000))
                    TEST_FILE=$(mktemp)

                    cleanup() {
                        docker rm -f "$CONTAINER_NAME" >/dev/null 2>&1 || true
                        rm -f "$TEST_FILE"
                    }
                    trap cleanup EXIT

                    docker run -d \
                        --name "$CONTAINER_NAME" \
                        -p "127.0.0.1:${TEST_PORT}:80" \
                        ${IMAGE_NAME}:${BUILD_NUMBER}

                    success=0

                    for attempt in $(seq 1 15); do
                        if curl --connect-timeout 2 -fsS \
                            "http://127.0.0.1:${TEST_PORT}/" \
                            -o "$TEST_FILE"; then
                            success=1
                            break
                        fi

                        sleep 2
                    done

                    if [ "$success" -ne 1 ]; then
                        echo "Container HTTP test failed."
                        docker logs "$CONTAINER_NAME" || true
                        exit 1
                    fi

                    grep -qi '<html' "$TEST_FILE"
                    grep -qi '</html>' "$TEST_FILE"

                    echo "Container smoke test passed."
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

                        IMAGE="${DH_USER}/${IMAGE_NAME}"

                        echo "$DH_TOKEN" | docker login \
                            --username "$DH_USER" \
                            --password-stdin

                        cleanup() {
                            docker logout >/dev/null 2>&1 || true
                        }
                        trap cleanup EXIT

                        retry_push() {
                            image="$1"
                            attempt=1
                            max_attempts=3

                            while [ "$attempt" -le "$max_attempts" ]; do
                                echo "Pushing $image (attempt $attempt/$max_attempts)"

                                if docker push "$image"; then
                                    echo "Successfully pushed $image"
                                    return 0
                                fi

                                if [ "$attempt" -eq "$max_attempts" ]; then
                                    echo "Push failed after $max_attempts attempts."
                                    return 1
                                fi

                                delay=$((attempt * 10))
                                echo "Push failed; retrying in ${delay}s..."
                                sleep "$delay"
                                attempt=$((attempt + 1))
                            done
                        }

                        retry_push "${IMAGE}:${BUILD_NUMBER}"
                        retry_push "${IMAGE}:latest"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Portfolio image built, tested, and pushed to Docker Hub.'
        }

        failure {
            echo 'FAILURE: Inspect the failed stage in the Jenkins console log.'
        }

        always {
            echo "Build result: ${currentBuild.currentResult}"
            echo "Build number: ${BUILD_NUMBER}"
        }
    }
}
