
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        IMAGE_NAME = 'devops-portfolio'
        IMAGE_REPO = 'imaravinda/devops-portfolio'
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

                        docker tag "$LOCAL_IMAGE" "${REMOTE_IMAGE}:${BUILD_NUMBER}"
                        docker tag "$LOCAL_IMAGE" "${REMOTE_IMAGE}:latest"

                        echo "$DH_TOKEN" | docker login \
                            --username "$DH_USER" \
                            --password-stdin

                        docker push "${REMOTE_IMAGE}:${BUILD_NUMBER}"
                        docker push "${REMOTE_IMAGE}:latest"

                        docker logout
                    '''
                }
            }
        }

        stage('Update GitOps Manifest') {
            steps {
                dir('gitops') {
                    git branch: 'main',
                        credentialsId: 'github-gitops-token',
                        url: 'https://github.com/AravindaKumar-0/devops-portfolio-gitops.git'

                    sh '''
                        set -eu

                        sed -i "s#image: ${IMAGE_REPO}:.*#image: ${IMAGE_REPO}:${BUILD_NUMBER}#" \
                            k8s/deployment.yaml

                        git config user.name "Jenkins"
                        git config user.email "jenkins@localhost"

                        git add k8s/deployment.yaml

                        if git diff --cached --quiet; then
                            echo "No GitOps manifest changes."
                        else
                            git commit -m "Deploy portfolio image ${BUILD_NUMBER}"
                        fi
                    '''

                    withCredentials([
                        usernamePassword(
                            credentialsId: 'github-gitops-token',
                            usernameVariable: 'GIT_USER',
                            passwordVariable: 'GIT_TOKEN'
                        )
                    ]) {
                        sh '''
                            set -eu

                            git -c credential.helper='!f() { echo "username=$GIT_USER"; echo "password=$GIT_TOKEN"; }; f' \
                                push origin HEAD:main
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Image pushed to Docker Hub and GitOps deployment updated.'
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
