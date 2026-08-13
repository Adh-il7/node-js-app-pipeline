pipeline {

    agent {
        docker {
            image 'node:20-alpine'
            args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    options {
        skipDefaultCheckout(true)
    }

    stages {

        stage('Install Tools') {
            steps {
                sh '''
                    echo "Installing required tools..."

                    apk add --no-cache \
                        git \
                        docker-cli \
                        bash \
                        curl

                    echo "Git version:"
                    git --version

                    echo "Node version:"
                    node --version

                    echo "NPM version:"
                    npm --version

                    echo "Docker version:"
                    docker --version
                '''
            }
        }

        stage('Checkout') {
            steps {
                checkout scm

                sh '''
                    echo "======================================"
                    echo "Checkout completed successfully"
                    echo "======================================"

                    echo "Current directory:"
                    pwd

                    echo "Project files:"
                    ls -la

                    echo "node-app directory:"
                    ls -la node-app
                '''
            }
        }

        stage('Build and Test') {
            steps {
                sh '''
                    cd node-app

                    echo "======================================"
                    echo "Installing Node.js dependencies"
                    echo "======================================"

                    npm ci

                    echo "======================================"
                    echo "Running tests"
                    echo "======================================"

                    npm test
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {

                    sh '''
                        cd node-app

                        echo "======================================"
                        echo "Installing SonarQube Scanner"
                        echo "======================================"

                        npm install --save-dev sonar-scanner

                        echo "Fixing scanner permissions..."

                        chmod +x node_modules/sonar-scanner/bin/sonar-scanner

                        echo "======================================"
                        echo "Running SonarQube Analysis"
                        echo "======================================"

                        ./node_modules/sonar-scanner/bin/sonar-scanner \
                          -Dsonar.projectKey=node-express-app \
                          -Dsonar.projectName="Node Express App" \
                          -Dsonar.sources=. \
                          -Dsonar.exclusions=node_modules/**,coverage/** \
                          -Dsonar.host.url="$SONAR_HOST_URL"

                        echo "======================================"
                        echo "SonarQube Analysis Completed"
                        echo "======================================"
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            environment {
                DOCKER_IMAGE = "adhil7/node-js-app:${BUILD_NUMBER}"
            }

            steps {
                sh '''
                    echo "======================================"
                    echo "Building Docker Image"
                    echo "======================================"

                    docker build \
                      -t ${DOCKER_IMAGE} \
                      node-app

                    echo "Docker image created:"
                    docker images | grep node-js-app || true
                '''
            }
        }

        stage('Push Docker Image') {
            environment {
                DOCKER_IMAGE = "adhil7/node-js-app:${BUILD_NUMBER}"
            }

            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-cred',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {

                    sh '''
                        echo "======================================"
                        echo "Logging into Docker Hub"
                        echo "======================================"

                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin

                        echo "Pushing build image..."

                        docker push ${DOCKER_IMAGE}

                        echo "Tagging latest image..."

                        docker tag \
                            ${DOCKER_IMAGE} \
                            adhil7/node-js-app:latest

                        echo "Pushing latest image..."

                        docker push adhil7/node-js-app:latest

                        echo "Logging out from Docker Hub..."

                        docker logout
                    '''
                }
            }
        }

        stage('Update Deployment File') {

            environment {
                GIT_REPO_NAME = "node-js-app-pipeline"
                GIT_USER_NAME = "Ad-hil7"
            }

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'github',
                        usernameVariable: 'GITHUB_USERNAME',
                        passwordVariable: 'GITHUB_TOKEN'
                    )
                ]) {

                    sh '''
                        echo "======================================"
                        echo "Cloning Deployment Repository"
                        echo "======================================"

                        rm -rf repo-temp

                        git clone \
                          https://${GITHUB_USERNAME}:${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git \
                          repo-temp

                        cd repo-temp

                        echo "Repository cloned successfully"

                        git config user.email "adhilstar303@gmail.com"
                        git config user.name "${GIT_USER_NAME}"

                        echo "======================================"
                        echo "Updating Kubernetes Deployment"
                        echo "======================================"

                        sed -i \
                          "s|image: .*|image: adhil7/node-js-app:${BUILD_NUMBER}|g" \
                          node-app-manifests/deployment.yml

                        echo "Updated deployment file:"
                        cat node-app-manifests/deployment.yml

                        echo "======================================"
                        echo "Committing changes"
                        echo "======================================"

                        git add node-app-manifests/deployment.yml

                        git commit \
                          -m "Update node app image tag to ${BUILD_NUMBER} [skip ci]" \
                          || echo "No changes to commit"

                        echo "======================================"
                        echo "Pushing changes to GitHub"
                        echo "======================================"

                        git push origin main

                        echo "Deployment file updated successfully"
                    '''
                }
            }
        }
    }

    post {

        success {
            echo '======================================'
            echo 'PIPELINE COMPLETED SUCCESSFULLY'
            echo '======================================'
        }

        failure {
            echo '======================================'
            echo 'PIPELINE FAILED'
            echo 'Check the failed stage above'
            echo '======================================'
        }

        always {
            sh '''
                echo "Cleaning
