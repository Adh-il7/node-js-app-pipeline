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
                    apk add --no-cache git docker-cli bash curl
                    echo "Git:"
                    git --version
                    echo "Node:"
                    node --version
                    echo "NPM:"
                    npm --version
                    echo "Docker:"
                    docker --version
                '''
            }
        }

        stage('Checkout') {
            steps {
                checkout scm

                sh '''
                    echo "Checkout completed"
                    echo "Current directory:"
                    pwd

                    echo "Files:"
                    ls -la

                    echo "node-app:"
                    ls -la node-app
                '''
            }
        }

        stage('Build and Test') {
            steps {
                sh '''
                    cd node-app

                    echo "Installing dependencies..."
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

                        chmod +x node_modules/sonar-scanner/bin/sonar-scanner

                        echo "Running SonarQube analysis..."

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

        stage('Build Docker Image') {
            environment {
                DOCKER_IMAGE = "adhil7/node-js-app:${BUILD_NUMBER}"
            }

            steps {
                sh '''
                    echo "Building Docker image..."

                    docker build \
                        -t "$DOCKER_IMAGE" \
                        node-app

                    docker images
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
                        echo "Logging into Docker Hub..."

                        echo "$DOCKER_PASSWORD" | docker login \
                            --username "$DOCKER_USERNAME" \
                            --password-stdin

                        echo "Pushing image..."

                        docker push "$DOCKER_IMAGE"

                        echo "Creating latest tag..."

                        docker tag \
                            "$DOCKER_IMAGE" \
                            "adhil7/node-js-app:latest"

                        echo "Pushing latest..."

                        docker push "adhil7/node-js-app:latest"

                        docker logout
                    '''
                }
            }
        }

        stage('Update Deployment File') {
            environment {
                GIT_REPO_NAME = 'node-js-app-pipeline'
                GIT_USER_NAME = 'Ad-hil7'
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
                            "https://${GITHUB_USERNAME}:${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git" \
                            repo-temp

                        cd repo-temp

                        git config user.email "adhilstar303@gmail.com"
                        git config user.name "$GIT_USER_NAME"

                        echo "Updating deployment image..."

                        sed -i \
                            "s|image: .*|image: adhil7/node-js-app:${BUILD_NUMBER}|g" \
                            node-app-manifests/deployment.yml

                        echo "Deployment file:"
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

    post {
        success {
            echo 'PIPELINE COMPLETED SUCCESSFULLY'
        }

        failure {
            echo 'PIPELINE FAILED'
        }

        always {
            sh '''
                rm -rf repo-temp || true
            '''
        }
    }
}
