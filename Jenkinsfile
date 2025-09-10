pipeline {
  agent any
  environment {
    APP_NAME = "springboot-app"
    AWS_REGION = "us-east-1"
    DOCKER_REGISTRY = "<AWS_ACCOUNT_ID>.dkr.ecr.${AWS_REGION}.amazonaws.com"
    IMAGE = "${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER}"
    KUBE_CONFIG_CREDENTIALS_ID = "kubeconfig"   // Jenkins secret text ID
    AWS_CREDENTIALS_ID = "aws-creds"           // Jenkins AWS creds ID
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build Jar') {
      steps {
        sh 'mvn -B -DskipTests package'
      }
    }

    stage('Build & Push Docker Image') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: env.AWS_CREDENTIALS_ID]]) {
          sh '''
            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${DOCKER_REGISTRY}
            docker build -t ${IMAGE} .
            docker push ${IMAGE}
          '''
        }
      }
    }

    stage('Deploy to EKS') {
      steps {
        withCredentials([string(credentialsId: env.KUBE_CONFIG_CREDENTIALS_ID, variable: 'KUBECONF_DATA')]) {
          sh '''
            echo "$KUBECONF_DATA" > kubeconfig
            export KUBECONFIG=$PWD/kubeconfig
            kubectl apply -f k8s/deployment.yaml
            kubectl apply -f k8s/service.yaml
            kubectl set image deployment/${APP_NAME} ${APP_NAME}=${IMAGE} -n default --record
            kubectl rollout status deployment/${APP_NAME} -n default --timeout=120s
          '''
        }
      }
    }

    stage('Health Check') {
      steps {
        withCredentials([string(credentialsId: env.KUBE_CONFIG_CREDENTIALS_ID, variable: 'KUBECONF_DATA')]) {
          sh '''
            echo "$KUBECONF_DATA" > kubeconfig
            export KUBECONFIG=$PWD/kubeconfig
            SERVICE_IP=""
            for i in {1..30}; do
              SERVICE_IP=$(kubectl get svc ${APP_NAME} -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
              if [ -n "$SERVICE_IP" ]; then break; fi
              sleep 10
            done
            echo "App available at: http://$SERVICE_IP"
            curl -f http://$SERVICE_IP/actuator/health
          '''
        }
      }
    }
  }
}
