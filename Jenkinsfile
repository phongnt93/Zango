pipeline {
  agent {
    kubernetes {
      label 'kaniko-agent'
      defaultContainer 'jnlp'
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:latest
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: kaniko-secret
      mountPath: /kaniko/.docker
  volumes:
  - name: kaniko-secret
    secret:
      secretName: kaniko-secret
"""
    }
  }

  environment {
    HARBOR     = 'harbor.local'
    PROJECT    = 'zango'
    DEPLOY_NS  = 'zango'
    IMAGE_TAG  = "${env.BRANCH_NAME}-${env.BUILD_ID}"
    IMAGE      = "${HARBOR}/${PROJECT}/zango-app:${IMAGE_TAG}"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build & Push with Kaniko') {
      steps {
        container('kaniko') {
          sh """
            /kaniko/executor \
              --context=\$WORKSPACE \
              --dockerfile=\$WORKSPACE/Dockerfile \
              --destination=${IMAGE} \
              --cache=true
          """
        }
      }
    }

    stage('Wait for Harbor Scan') {
      steps {
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
          sh "kubectl set image deployment/zango zango=${IMAGE} -n ${DEPLOY_NS}"
        }
      }
    }
  }

  post {
    success {
      echo "✅ Deployed ${IMAGE} successfully"
    }
    failure {
      echo "❌ Pipeline failed—check logs!"
    }
  }
}

