pipeline {
  agent any

  environment {
    IMAGE_TAG = "${BUILD_NUMBER}"
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
          sh '''
          echo "===== Backend Sonar Scan ====="
          cd backend
          sonar-scanner -Dsonar.projectKey=chatapp-backend -Dsonar.sources=. -Dsonar.language=js

          cd ..

          echo "===== Frontend Sonar Scan ====="
          cd frontend
          sonar-scanner \
            -Dsonar.projectKey=chatapp-frontend \
            -Dsonar.sources=. \
            -Dsonar.language=js
          '''
        }
      }
    }


    stage('Quality Gate') {
      steps {
        waitForQualityGate abortPipeline: true
      }
    }

    stage('Build Docker Images') {
      steps {
        sh '''
        docker build -t ${BACKEND_IMAGE}:${IMAGE_TAG} backend/
        docker build -t ${FRONTEND_IMAGE}:${IMAGE_TAG} frontend/
        '''
      }
    }

    stage('Push Docker Images') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-creds',
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          sh '''
          echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin

          docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
          docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
          '''
        }
      }
    }

    stage('Update Kubernetes Manifests') {
      steps {
        sh '''
        sed -i 's|image: .*chatapp-backend:.*|image: xerox2/chatapp-backend:'"${IMAGE_TAG}"'|g' k8s/backend-deployment.yaml
        sed -i 's|image: .*chatapp-frontend:.*|image: xerox2/chatapp-frontend:'"${IMAGE_TAG}"'|g' k8s/frontend-deployment.yaml
        '''
      }
    }

    stage('Git Commit (GitOps)') {
      steps {
        sh '''
        git config user.name "Pushpendra Singh"
        git config user.email "pushpendrasingh0549@gmail.com"
        git add k8s/
        git commit -m "Deploy FE + BE image tag ${IMAGE_TAG}"
        git push origin main
        '''
      }
    }
  }
}
