pipeline {
  agent any

  environment {
    HARBOR    = 'harbor.local'
    PROJECT   = 'zango'
    DEPLOY_NS = 'zango'
    IMAGE_TAG = "${env.BRANCH_NAME}-${env.BUILD_ID}"
    IMAGE     = "${HARBOR}/${PROJECT}/zango-app:${IMAGE_TAG}"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build Docker Image') {
      steps {
        // Dùng double‑quoted string và env.IMAGE
        sh "docker build -t ${env.IMAGE} ."
      }
    }

    stage('Run Tests') {
      steps {
        sh "docker run --rm ${env.IMAGE} python manage.py test"
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
          // Login và push
          sh """
            echo \$H_PASS | docker login ${env.HARBOR} -u \$H_USER --password-stdin
            docker push ${env.IMAGE}
          """
        }
      }
    }

    stage('Wait for Harbor Scan') {
      steps {
        waitForHarborWebHook abortPipeline: true,
                            credentialsId: 'harbor-robot',
                            server: "${env.HARBOR}",
                            fullImageName: "${env.IMAGE}",
                            severity: 'Medium'
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        withCredentials([kubeconfigFile(credentialsId: 'orbstack-kubeconfig', variable: 'KUBECONFIG')]) {
          sh "kubectl set image deployment/zango zango=${env.IMAGE} -n ${env.DEPLOY_NS}"
        }
      }
    }
  }

  post {
    always {
      echo ">>> Build finished: ${currentBuild.currentResult}"
    }
    success {
      echo "✅ SUCCESS: ${env.IMAGE} pushed & deployed."
    }
    failure {
      echo "❌ FAILURE: Check above logs."
    }
  }
}

