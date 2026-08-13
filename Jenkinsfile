pipeline {
    agent {
        docker {
            image 'node:20-alpine'
            args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    stages {

        stage('Checkout') {
            steps {
                sh '''
                    echo "Starting build process..."
                    echo "Node version:"
                    node --version
                    echo "NPM version:"
                    npm --version
                '''
            }
        }

        stage('Install Tools') {
            steps {
                sh '''
                    echo "Installing required tools..."

                    apk add --no-cache \
                        git \
                        docker-cli \
                        bash

                    echo "Docker version:"
                    docker --version

                    echo "Git version:"
                    git --version
                '''
            }
        }

        stage('Build and Test') {
            steps {
                sh '''
                    cd node-app

                    echo "Installing Node dependencies..."
                    npm ci

                    echo "Running tests..."
                    npm test
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''
                        cd node-app

                        echo "Installing SonarQube Scanner..."

                        npm install --save-dev sonar-scanner

                        echo "Fixing SonarQube Scanner permissions..."

                        chmod +x node_modules/sonar-scanner/bin/sonar-scanner

                        echo "Running SonarQube Analysis..."

                        ./node_modules/sonar-scanner/bin/sonar-scanner \
                          -Dsonar.projectKey=node-express-app \
                          -Dsonar.projectName="Node Express App" \
                          -Dsonar.sources=. \
                          -Dsonar.exclusions=node_modules/**,coverage/** \
                          -Dsonar.host.url=$SONAR_HOST_URL

                        echo "SonarQube analysis completed."
                    '''
                }
            }
        }

        stage('Build and Push Docker Image') {
            environment {
                DOCKER_IMAGE = "adhil7/node-js-app:${BUILD_NUMBER}"
            }

            steps {
                script {

                    echo "Building Docker image..."

                    sh '''
                        docker build \
                          -t ${DOCKER_IMAGE} \
                          node-app
                    '''

                    echo "Pushing Docker image..."

                    def dockerImage = docker.image("${DOCKER_IMAGE}")

                    docker.withRegistry(
                        'https://index.docker.io/v1/',
                        'docker-cred'
                    ) {
                        dockerImage.push()
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Update Deployment File') {
            environment {
                GIT_REPO_NAME = "node-js-app-pipeline"
                GIT_USER_NAME = "Adh-il7"
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
                        echo "Cloning deployment repository..."

                        rm -rf repo-temp

                        git clone \
                          https://${GITHUB_USERNAME}:${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git \
                          repo-temp

                        cd repo-temp

                        git config user.email "adhilstar303@gmail.com"
                        git config user.name "${GIT_USER_NAME}"

                        echo "Updating Kubernetes deployment image..."

                        sed -i \
                          "s|image: .*|image: adhil7/node-js-app:${BUILD_NUMBER}|g" \
                          node-app-manifests/deployment.yml

                        echo "Updated deployment:"
                        cat node-app-manifests/deployment.yml

                        git add node-app-manifests/deployment.yml

                        git commit \
                          -m "Update node app image tag to ${BUILD_NUMBER} [skip ci]" \
                          || echo "No changes to commit"

                        git push origin main
                    '''
                }
            }
        }
    }
}
