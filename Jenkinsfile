pipeline {
  agent {
    kubernetes {
      label 'kaniko-agent'
      defaultContainer 'kaniko'
      yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: default
  containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:latest
      args:
        - --cache=true
        - --context=\$(WORKSPACE)
        - --dockerfile=\$(WORKSPACE)/Dockerfile
        - --destination=\$(HARBOR)/\$(PROJECT)/zango-app:\$(IMAGE_TAG)
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
          sh '/kaniko/executor'
        }
      }
    }

    stage('Wait for Harbor Scan') {
      steps {
        waitForHarborWebHook abortPipeline: true,
                           credentialsId: 'harbor-robot',
                           server: "${HARBOR}",
                           fullImageName: "${HARBOR}/${PROJECT}/zango-app:${IMAGE_TAG}",
                           severity: 'Medium'
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        withCredentials([kubeconfigFile(credentialsId: 'orbstack-kubeconfig', variable: 'KUBECONFIG')]) {
          sh """
            kubectl set image deployment/zango \
              zango=${HARBOR}/${PROJECT}/zango-app:${IMAGE_TAG} \
              -n ${DEPLOY_NS}
          """
        }
      }
    }
  }

  post {
    success {
      echo "✅ Deployed ${HARBOR}/${PROJECT}/zango-app:${IMAGE_TAG} successfully"
    }
    failure {
      echo "❌ Pipeline failed, check above logs"
    }
  }
}

