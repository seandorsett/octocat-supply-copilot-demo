pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
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
                // Ensure a clean workspace and checkout from Git
                deleteDir()
                checkout scm
            }
        }

        stage('Install & Build Node App') {
            steps {
                sh '''
                    set -eu

                    if ! command -v node >/dev/null 2>&1; then
                        NODE_VERSION="v20.19.0"
                        NODE_DIST="node-${NODE_VERSION}-linux-x64"
                        mkdir -p .tools

                        if [ ! -x ".tools/${NODE_DIST}/bin/node" ]; then
                            rm -rf ".tools/${NODE_DIST}" ".tools/${NODE_DIST}.tar.xz"
                            if command -v curl >/dev/null 2>&1; then
                                curl -fsSL "https://nodejs.org/dist/${NODE_VERSION}/${NODE_DIST}.tar.xz" -o ".tools/${NODE_DIST}.tar.xz"
                            elif command -v wget >/dev/null 2>&1; then
                                wget -q "https://nodejs.org/dist/${NODE_VERSION}/${NODE_DIST}.tar.xz" -O ".tools/${NODE_DIST}.tar.xz"
                            else
                                echo "Neither curl nor wget is available to download Node.js"
                                exit 1
                            fi

                            tar -xJf ".tools/${NODE_DIST}.tar.xz" -C .tools
                        fi

                        export PATH="$PWD/.tools/${NODE_DIST}/bin:$PATH"
                    fi

                    node --version
                    npm --version
                    npm install
                    npm run build
                '''
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

        stage('Deploy to Azure Web App') {
            steps {
                // Combine Azure SP + ACR creds
                withCredentials([
                    usernamePassword(
                        credentialsId: 'azure-sp',
                        usernameVariable: 'AZ_CLIENT_ID',
                        passwordVariable: 'AZ_CLIENT_SECRET'
                    ),
                    usernamePassword(
                        credentialsId: 'acr-creds',
                        usernameVariable: 'ACR_USER',
                        passwordVariable: 'ACR_PASS'
                    )
                ]) {
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

    post {
        always {
            echo "Pipeline finished."
        }
        failure {
            echo "Pipeline failed! Check console output for errors."
        }
    }
}
