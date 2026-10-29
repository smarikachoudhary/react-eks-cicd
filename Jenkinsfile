pipeline {
  agent any

  parameters {
    booleanParam(name: 'DEPLOY_TO_EKS', defaultValue: false,
                 description: 'EKS cluster ready hone ke baad hi true karo')
  }

  environment {
    IMAGE_NAME            = 'smarika256/react-eks-cicd'
    IMAGE_TAG             = "${BUILD_NUMBER}"
    AWS_DEFAULT_REGION    = 'ap-south-1'
    CLUSTER_NAME          = 'react-eks-cluster'
    AWS_ACCESS_KEY_ID     = credentials('aws-access-key-id')
    AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
  }

  stages {

    stage('Clean Workspace') {
      steps {
        deleteDir()
      }
    }

    stage('Checkout') {
      steps {
        git branch: 'main', url: 'https://github.com/smarikachoudhary/react-eks-cicd.git'
      }
    }

    stage('Install Dependencies') {
      steps {
        sh 'npm ci'
      }
    }

    stage('Build') {
      steps {
        sh 'npm run build'
      }
    }

    stage('Test') {
      steps {
        sh 'npm test'
      }
    }

    stage('SonarQube Scan') {
      steps {
        script {
          def scannerHome = tool 'sonar-scanner'
          withSonarQubeEnv('sonarqube') {
            sh "${scannerHome}/bin/sonar-scanner"
          }
        }
      }
    }

    stage('Quality Gate') {
      steps {
        timeout(time: 5, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('Docker Build') {
      steps {
        sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -t ${IMAGE_NAME}:latest ."
      }
    }

    stage('Trivy Scan') {
      steps {
        sh "trivy image --severity HIGH,CRITICAL --exit-code 0 --no-progress --format table -o trivy-report.txt ${IMAGE_NAME}:${IMAGE_TAG}"
        sh 'cat trivy-report.txt'
        archiveArtifacts artifacts: 'trivy-report.txt', allowEmptyArchive: true
      }
    }

    stage('Push Docker Image') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                          usernameVariable: 'DH_USER',
                                          passwordVariable: 'DH_PASS')]) {
          sh 'echo $DH_PASS | docker login -u $DH_USER --password-stdin'
          sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
          sh "docker push ${IMAGE_NAME}:latest"
        }
      }
    }

    stage('Connect to EKS') {
      when { expression { return params.DEPLOY_TO_EKS } }
      steps {
        sh 'aws sts get-caller-identity'
        sh 'aws eks update-kubeconfig --region $AWS_DEFAULT_REGION --name $CLUSTER_NAME'
        sh 'kubectl get nodes'
      }
    }

    stage('Deploy to Kubernetes') {
      when { expression { return params.DEPLOY_TO_EKS } }
      steps {
        sh "sed -i 's|image: .*|image: ${IMAGE_NAME}:${IMAGE_TAG}|' k8s/deployment.yaml"
        sh 'kubectl apply -f k8s/'
      }
    }

    stage('Verify Rollout') {
      when { expression { return params.DEPLOY_TO_EKS } }
      steps {
        sh 'kubectl rollout status deployment/react-app --timeout=180s'
        sh 'kubectl get pods -o wide'
        sh 'kubectl get svc react-app-service'
      }
    }

    stage('Health Check') {
      when { expression { return params.DEPLOY_TO_EKS } }
      steps {
        sh '''
          HOST=""
          for i in $(seq 1 30); do
            HOST=$(kubectl get svc react-app-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
            if [ -n "$HOST" ]; then break; fi
            echo "LoadBalancer ka URL abhi ban raha hai..."
            sleep 10
          done
          echo "App URL: http://$HOST"
          for i in $(seq 1 30); do
            CODE=$(curl -s -o /dev/null -w "%{http_code}" http://$HOST || true)
            echo "HTTP status: $CODE"
            if [ "$CODE" = "200" ]; then
              echo "Health check PASS"
              exit 0
            fi
            sleep 10
          done
          echo "Health check FAIL"
          exit 1
        '''
      }
    }
  }

  post {
    always {
      sh 'docker logout || true'
      sh 'docker image prune -f || true'
    }
    success {
      echo 'Pipeline SUCCESS'
    }
    failure {
      echo 'Pipeline FAILED - Console Output dekho'
    }
  }
}