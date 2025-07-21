pipeline {
  agent any

  environment {
    IMAGE = "zango-app:${env.BRANCH_NAME}-${env.BUILD_ID}"
    REGISTRY = "" // nếu bạn dùng Harbor, ví dụ: harbor.local/project
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build Docker Image') {
      steps {
        sh 'docker build -t ${IMAGE} .'
      }
    }

    stage('Run Tests') {
      steps {
        sh 'docker run --rm ${IMAGE} python manage.py test'
      }
    }

    stage('Push to Registry') {
      when { expression { env.REGISTRY != "" } }
      steps {
        withCredentials([usernamePassword(credentialsId: 'harbor-robot', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
          sh """
            docker tag ${IMAGE} ${REGISTRY}/${IMAGE}
            echo $PASS | docker login ${REGISTRY} -u $USER --password-stdin
            docker push ${REGISTRY}/${IMAGE}
          """
        }
      }
    }

    stage('Deploy to K8s') {
      when { expression { env.REGISTRY != "" } }
      steps {
        sh """
          kubectl -n your-namespace set image deployment/zango zango=${REGISTRY}/${IMAGE}
        """
      }
    }
  }

  post {
    always {
      echo "Build finished: ${currentBuild.currentResult}"
    }
    success { echo "✅ Success!" }
    failure { echo "❌ Failed!" }
  }
}

