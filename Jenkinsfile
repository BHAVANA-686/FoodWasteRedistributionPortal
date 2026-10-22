
pipeline {
    agent any

    parameters {
        choice(
            name: 'DEPLOY_ENV',
            choices: ['staging', 'production'],
            description: 'Select the deployment environment'
        )
    }

    environment {
        IMAGE_NAME = 'bhavanaraman/food-waste-portal'
        IMAGE_TAG = 'v1'
        K8S_DEPLOYMENT = 'food-waste-portal'
        K8S_CONTAINER = 'food-waste-portal'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate Application') {
            steps {
                script {
                    if (!fileExists('index.html')) {
                        error('index.html is missing')
                    }

                    if (!fileExists('Dockerfile')) {
                        error('Dockerfile is missing')
                    }

                    if (!fileExists('deployment.yaml')) {
                        error('deployment.yaml is missing')
                    }

                    echo 'Required application files are present.'
                }
            }
        }

        stage('Parallel Validation') {
            parallel {

                stage('HTML Validation') {
                    steps {
                        script {
                            def html = readFile('index.html')

                            if (!html.contains('<html')) {
                                error('HTML opening tag is missing')
                            }

                            if (!html.contains('foodRecords')) {
                                error('Food records section is missing')
                            }

                            echo 'HTML validation passed.'
                        }
                    }
                }

                stage('Dockerfile Validation') {
                    steps {
                        script {
                            def dockerfile = readFile('Dockerfile')

                            if (!dockerfile.contains('FROM nginx:alpine')) {
                                error('Expected Nginx base image is missing')
                            }

                            echo 'Dockerfile validation passed.'
                        }
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'dockerhub-credentials',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )
                    ]) {
                        sh '''
                            set -eu
                            echo "$DOCKER_PASS" | docker login \
                              -u "$DOCKER_USER" --password-stdin
                            docker build -t "$IMAGE_NAME:$IMAGE_TAG" .
                            docker logout
                        '''
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'dockerhub-credentials',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )
                    ]) {
                        sh '''
                            set -eu
                            echo "$DOCKER_PASS" | docker login \
                              -u "$DOCKER_USER" --password-stdin
                            docker push "$IMAGE_NAME:$IMAGE_TAG"
                            docker logout
                        '''
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            when {
                expression {
                    params.DEPLOY_ENV == 'staging'
                }
            }

            steps {
                script {
                    withCredentials([
                        file(
                            credentialsId: 'kubeconfig',
                            variable: 'KUBECONFIG'
                        )
                    ]) {
                        sh '''
                            set -eu
                            kubectl apply -f deployment.yaml

                            kubectl set image deployment/$K8S_DEPLOYMENT \
                              $K8S_CONTAINER=$IMAGE_NAME:$IMAGE_TAG

                            if ! kubectl rollout status \
                              deployment/$K8S_DEPLOYMENT --timeout=120s; then
                                kubectl rollout undo \
                                  deployment/$K8S_DEPLOYMENT
                                kubectl rollout status \
                                  deployment/$K8S_DEPLOYMENT --timeout=120s
                                exit 1
                            fi

                            kubectl get deployments
                            kubectl get pods
                            kubectl get services
                        '''
                    }
                }
            }
        }

        stage('Archive Build Artifacts') {
            steps {
                archiveArtifacts(
                    artifacts: 'index.html,Dockerfile,deployment.yaml,Jenkinsfile',
                    fingerprint: true
                )
            }
        }
    }

    post {
        success {
            echo 'Food Waste Portal pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Review the console output.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}

