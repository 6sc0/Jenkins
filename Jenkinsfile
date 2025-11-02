pipeline {
  agent any

  environment {
    DOCKER_ID = "6sc0" // Ton DockerHub ID
    DOCKER_IMAGE = "datascientestapi"
    DOCKER_TAG = "v.${BUILD_ID}.0"
  }

  stages {

    stage('Docker Clean & Build') {
      steps {
        script {
          sh '''
            echo "[INFO] Cleaning existing container if any..."
            docker rm -f jenkins || true
            echo "[INFO] Building Docker image..."
            docker build -t $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG .
          '''
        }
      }
    }

    stage('Docker Run for Test') {
      steps {
        script {
          sh '''
            echo "[INFO] Running container for test..."
            docker run -d -p 8083:80 --name jenkins $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG || (echo "[ERROR] Port 80 already in use. Using 8080 instead." && exit 1)
            sleep 5
          '''
        }
      }
    }

    stage('Test Acceptance') {
      steps {
        script {
          sh '''
            echo "[INFO] Performing health check..."
            curl --fail http://localhost:8083 || (echo "[ERROR] Curl test failed!" && docker logs jenkins && exit 1)
          '''
        }
      }
    }

    stage('Docker Push') {
      environment {
        DOCKER_PASS = credentials("DOCKER_HUB_PASS")
      }
      steps {
        script {
          sh '''
            echo "[INFO] Logging into DockerHub..."
            echo $DOCKER_PASS | docker login -u $DOCKER_ID --password-stdin
            echo "[INFO] Pushing Docker image..."
            docker push $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG
          '''
        }
      }
    }

    stage('Deploy to Dev') {
      environment {
        KUBECONFIG = credentials("config")
      }
      steps {
        script {
          sh '''
            echo "[INFO] Preparing kubeconfig for dev..."
            mkdir -p .kube
            echo "$KUBECONFIG" > .kube/config

            cp fastapi/values.yaml values.yml || (echo "[ERROR] fastapi/values.yaml not found!" && exit 1)
            sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml

            echo "[INFO] Deploying to dev..."
            helm upgrade --install app fastapi --values=values.yml --namespace dev || (echo "[ERROR] Helm deployment failed" && exit 1)
          '''
        }
      }
    }

    stage('Deploy to Staging') {
      environment {
        KUBECONFIG = credentials("config")
      }
      steps {
        script {
          sh '''
            echo "[INFO] Preparing kubeconfig for staging..."
            mkdir -p .kube
            echo "$KUBECONFIG" > .kube/config

            cp fastapi/values.yaml values.yml || (echo "[ERROR] fastapi/values.yaml not found!" && exit 1)
            sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml

            echo "[INFO] Deploying to staging..."
            helm upgrade --install app fastapi --values=values.yml --namespace staging || (echo "[ERROR] Helm deployment failed" && exit 1)
          '''
        }
      }
    }

    stage('Deploy to Prod') {
      environment {
        KUBECONFIG = credentials("config")
      }
      steps {
        timeout(time: 15, unit: "MINUTES") {
          input message: 'Do you want to deploy in production ?', ok: 'Yes'
        }

        script {
          sh '''
            echo "[INFO] Preparing kubeconfig for prod..."
            mkdir -p .kube
            echo "$KUBECONFIG" > .kube/config

            cp fastapi/values.yaml values.yml || (echo "[ERROR] fastapi/values.yaml not found!" && exit 1)
            sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml

            echo "[INFO] Deploying to prod..."
            helm upgrade --install app fastapi --values=values.yml --namespace prod || (echo "[ERROR] Helm deployment failed" && exit 1)
          '''
        }
      }
    }
  }

  post {
    always {
      script {
        echo "[CLEANUP] Cleaning up test container"
        sh "docker rm -f jenkins || true"
      }
    }
    failure {
      echo "[FAILURE] Pipeline failed. Check logs above"
    }
  }
}
