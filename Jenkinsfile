pipeline {
  agent any

  environment {
    AWS_REGION   = 'us-east-1'
    ECR_REGISTRY = '768286545498.dkr.ecr.us-east-1.amazonaws.com'
    IMAGE_REPO   = 'wms/backend'
    IMAGE_TAG    = "${env.BUILD_NUMBER}"
    GITOPS_REPO  = 'github.com/KaveenAlahapperuma/water-management-system-gitops.git'
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build image') {
      steps {
        sh 'docker build -t $ECR_REGISTRY/$IMAGE_REPO:$IMAGE_TAG .'
      }
    }

    stage('Trivy scan') {
      steps {
        sh '''
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
            aquasec/trivy:latest image --severity HIGH,CRITICAL --exit-code 0 --no-progress \
            $ECR_REGISTRY/$IMAGE_REPO:$IMAGE_TAG
        '''
      }
    }

    stage('Push to ECR') {
      steps {
        sh '''
          aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
          docker push $ECR_REGISTRY/$IMAGE_REPO:$IMAGE_TAG
        '''
      }
    }

    stage('Update GitOps repo') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
          sh '''
            rm -rf gitops
            git clone https://$GIT_USER:$GIT_TOKEN@$GITOPS_REPO gitops
            cd gitops
            sed -i "s|image: .*wms/backend:.*|image: $ECR_REGISTRY/$IMAGE_REPO:$IMAGE_TAG|" k8s/backend-deployment.yaml
            git config user.email "jenkins@wms.local"
            git config user.name "Jenkins CI"
            git add k8s/backend-deployment.yaml
            git commit -m "Update backend image to $IMAGE_TAG" || echo "nothing to commit"
            git push origin main
          '''
        }
      }
    }
  }

  post {
    always { sh 'docker image prune -f || true' }
  }
}
