pipeline {
    agent any

    environment {
        // 'Sonar' should match the name in Jenkins Global Tool Configuration
        SONAR_SCANNER  = tool 'Sonar'
        IMAGE_TAG      = "${BUILD_NUMBER}"
        BACKEND_IMAGE  = "xerox2/chatapp-backend"
        FRONTEND_IMAGE = "xerox2/chatapp-frontend"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Sonar Scan - Backend & Frontend') {
            steps {
                withSonarQubeEnv('Sonar') {
                    sh """
                        echo "===== Backend Sonar Scan ====="
                        cd backend
                        ${SONAR_SCANNER}/bin/sonar-scanner \
                            -Dsonar.projectKey=chatapp-backend \
                            -Dsonar.sources=.

                        echo "===== Frontend Sonar Scan ====="
                        cd ../frontend
                        ${SONAR_SCANNER}/bin/sonar-scanner \
                            -Dsonar.projectKey=chatapp-frontend \
                            -Dsonar.sources=.
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                sh """
                    docker build -t ${BACKEND_IMAGE}:${IMAGE_TAG} backend/
                    docker build -t ${FRONTEND_IMAGE}:${IMAGE_TAG} frontend/
                """
            }
        }

        stage('Push Docker Images') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds', 
                    usernameVariable: 'DOCKER_USER', 
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin
                        docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
                        docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Update Kubernetes Manifests') {
            steps {
                sh """
                    sed -i 's|image: .*chatapp-backend:.*|image: ${BACKEND_IMAGE}:${IMAGE_TAG}|g' k8s/backend-deployment.yml
                    sed -i 's|image: .*chatapp-frontend:.*|image: ${FRONTEND_IMAGE}:${IMAGE_TAG}|g' k8s/frontend-deployment.yml
                """
            }
        }

        stage('Git Commit (GitOps)') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                    sh """
                        git config user.name "Pushpendra Singh"
                        git config user.email "pushpendrasingh0549@gmail.com"
                        
                        git remote set-url origin https://${GITHUB_TOKEN}@github.com/pushpendra-singh1176/full-stack_chatApp.git
                        
                        git add k8s/*.yml
                        git commit -m "Deploy FE + BE image tag ${IMAGE_TAG}" || echo "No changes to commit"
                        git push origin HEAD:main
                    """
                }
            }
        }
    }

    post {
        always {
            sh "docker logout || true"
        }
        success {
            echo "CI Pipeline Successful! Now triggering CD Pipeline..."
            build job: 'ChatApp-CD', wait: false
        }
        failure {
            echo "Pipeline Failed - CD pipeline will not be triggered ! Check logs for errors."
        }
    }
}
