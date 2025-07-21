pipeline {
  agent any

  environment {
    HARBOR       = 'harbor.local'
    PROJECT      = 'zango'
    DEPLOY_NS    = 'zango'
    IMAGE_TAG    = "${env.BRANCH_NAME}-${env.BUILD_ID}"
    IMAGE        = "${HARBOR}/${PROJECT}/zango-app:${IMAGE_TAG}"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build Docker Image') {
      steps {
        sh "docker build -t ${IMAGE} ."
      }
    }

    stage('Run Tests') {
      steps {
        sh "docker run --rm ${IMAGE} python manage.py test"
      }
    }

    stage('Push to Harbor') {
      steps {
        withCredentials([
          usernamePassword(
            credentialsId: 'harbor-robot',
            usernameVariable: 'H_USER',
            passwordVariable: 'H_PASS'
          )
        ]) {
          sh """
            echo \$H_PASS | docker login ${HARBOR} -u \$H_USER --password-stdin
            docker push ${IMAGE}
          """
        }
      }
    }

    stage('Wait for Harbor Scan') {
      steps {
        // Nếu scan trả mức ≥ Medium thì abort pipeline
        waitForHarborWebHook abortPipeline: true,
                            credentialsId: 'harbor-robot',
                            server: "${HARBOR}",
                            fullImageName: "${IMAGE}",
                            severity: 'Medium'
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        withCredentials([kubeconfigFile(credentialsId: 'orbstack-kubeconfig', variable: 'KUBECONFIG')]) {
          sh """
            kubectl set image deployment/zango zango=${IMAGE} -n ${DEPLOY_NS}
          """
        }
      }
    }
  }

  post {
    always {
      echo ">>> Build finished: ${currentBuild.currentResult}"
    }
    success {
      echo "✅ SUCCESS: ${IMAGE} pushed & deployed."
    }
    failure {
      echo "❌ FAILURE: Check logs above."
    }
  }
}

