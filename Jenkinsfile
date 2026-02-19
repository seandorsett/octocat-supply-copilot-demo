pipeline {
    agent {
        docker {
            image 'node:20'
        }

  environment {
    ACR_NAME       = 'octocatacr'
    IMAGE_NAME     = 'octocat-supply'
    IMAGE_TAG      = "${BUILD_NUMBER}"
    RESOURCE_GROUP = 'octocat-rg'
    WEBAPP_NAME    = 'octocat-supply-demo'
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Install & Build') {
      steps {
        sh 'npm install'
        sh 'npm run build'
      }
    }

    stage('Build Docker Image') {
      steps {
        sh """
          docker build \
          -t ${ACR_NAME}.azurecr.io/${IMAGE_NAME}:${IMAGE_TAG} .
        """
      }
    }

    stage('Push Image to ACR') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'acr-creds',
          usernameVariable: 'ACR_USER',
          passwordVariable: 'ACR_PASS'
        )]) {
          sh """
            docker login ${ACR_NAME}.azurecr.io \
              -u $ACR_USER -p $ACR_PASS

            docker push ${ACR_NAME}.azurecr.io/${IMAGE_NAME}:${IMAGE_TAG}
          """
        }
      }
    }

    stage('Deploy to Azure') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'azure-sp',
          usernameVariable: 'AZ_CLIENT_ID',
          passwordVariable: 'AZ_CLIENT_SECRET'
        )]) {
          sh """
            az login --service-principal \
              -u $AZ_CLIENT_ID \
              -p $AZ_CLIENT_SECRET \
              --tenant <TENANT_ID>

            az webapp config container set \
              --resource-group ${RESOURCE_GROUP} \
              --name ${WEBAPP_NAME} \
              --docker-custom-image-name ${ACR_NAME}.azurecr.io/${IMAGE_NAME}:${IMAGE_TAG} \
              --docker-registry-server-url https://${ACR_NAME}.azurecr.io \
              --docker-registry-server-user $ACR_USER \
              --docker-registry-server-password $ACR_PASS
          """
        }
      }
    }
  }
}
