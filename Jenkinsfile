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
    command: ['cat']
    tty: true
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
      steps { checkout scm }
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

    // … các stage tiếp theo giữ nguyên …
  }
}

