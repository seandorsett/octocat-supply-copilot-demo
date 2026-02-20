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
        TENANT_ID      = 'b0d8e6c7-4f51-40f2-b45c-bae4fd843f5b'
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
                            rm -rf ".tools/${NODE_DIST}" ".tools/${NODE_DIST}.tar.gz"
                            if command -v curl >/dev/null 2>&1; then
                                curl -fsSL "https://nodejs.org/dist/${NODE_VERSION}/${NODE_DIST}.tar.gz" -o ".tools/${NODE_DIST}.tar.gz"
                            elif command -v wget >/dev/null 2>&1; then
                                wget -q "https://nodejs.org/dist/${NODE_VERSION}/${NODE_DIST}.tar.gz" -O ".tools/${NODE_DIST}.tar.gz"
                            else
                                echo "Neither curl nor wget is available to download Node.js"
                                exit 1
                            fi

                            tar -xzf ".tools/${NODE_DIST}.tar.gz" -C .tools
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

        stage('Build & Push Image in ACR') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'azure-sp',
                        usernameVariable: 'AZ_CLIENT_ID',
                        passwordVariable: 'AZ_CLIENT_SECRET'
                    )
                ]) {
                    sh '''
                        set -eu

                        AZ_BIN="$(command -v az 2>/dev/null || true)"

                        if ! command -v az >/dev/null 2>&1; then
                            if ! command -v python3 >/dev/null 2>&1; then
                                if command -v apt-get >/dev/null 2>&1 && [ "$(id -u)" -eq 0 ]; then
                                    rm -f /etc/apt/sources.list.d/azure-cli.list
                                    apt-get update
                                    apt-get install -y python3 python3-venv
                                else
                                    echo "Azure CLI is required, but python3 is not available and this agent cannot install packages."
                                    echo "Fix by using a Jenkins image that preinstalls azure-cli (recommended), or install python3+python3-venv in the agent image."
                                    exit 1
                                fi
                            fi

                            AZ_VENV="$HOME/.azcli-venv"
                            AZ_BIN="$AZ_VENV/bin/az"
                            if [ ! -x "$AZ_BIN" ]; then
                                rm -rf "$AZ_VENV"
                                if ! python3 -m venv "$AZ_VENV"; then
                                    if command -v apt-get >/dev/null 2>&1 && [ "$(id -u)" -eq 0 ]; then
                                        rm -f /etc/apt/sources.list.d/azure-cli.list
                                        apt-get update
                                        apt-get install -y python3-venv
                                        python3 -m venv "$AZ_VENV"
                                    else
                                        echo "python3-venv (with ensurepip) is required to install Azure CLI in an isolated environment."
                                        echo "Install python3-venv in the Jenkins agent image."
                                        exit 1
                                    fi
                                fi
                                "$AZ_VENV/bin/python" -m pip install --upgrade pip
                                "$AZ_VENV/bin/pip" install azure-cli
                            fi
                        fi

                        if [ -z "$AZ_BIN" ]; then
                            echo "Azure CLI executable could not be resolved after installation."
                            exit 1
                        fi

                        "$AZ_BIN" version

                        "$AZ_BIN" login --service-principal \
                            -u "$AZ_CLIENT_ID" \
                            -p "$AZ_CLIENT_SECRET" \
                            --tenant "$TENANT_ID"

                        "$AZ_BIN" acr build \
                            --registry "$ACR_NAME" \
                            --image "$IMAGE_NAME:$IMAGE_TAG" \
                            .
                    '''
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
                    sh '''
                        set -eu

                        AZ_BIN="$(command -v az 2>/dev/null || true)"

                        if ! command -v az >/dev/null 2>&1; then
                            if ! command -v python3 >/dev/null 2>&1; then
                                if command -v apt-get >/dev/null 2>&1 && [ "$(id -u)" -eq 0 ]; then
                                    rm -f /etc/apt/sources.list.d/azure-cli.list
                                    apt-get update
                                    apt-get install -y python3 python3-venv
                                else
                                    echo "Azure CLI is required, but python3 is not available and this agent cannot install packages."
                                    echo "Fix by using a Jenkins image that preinstalls azure-cli (recommended), or install python3+python3-venv in the agent image."
                                    exit 1
                                fi
                            fi

                            AZ_VENV="$HOME/.azcli-venv"
                            AZ_BIN="$AZ_VENV/bin/az"
                            if [ ! -x "$AZ_BIN" ]; then
                                rm -rf "$AZ_VENV"
                                if ! python3 -m venv "$AZ_VENV"; then
                                    if command -v apt-get >/dev/null 2>&1 && [ "$(id -u)" -eq 0 ]; then
                                        rm -f /etc/apt/sources.list.d/azure-cli.list
                                        apt-get update
                                        apt-get install -y python3-venv
                                        python3 -m venv "$AZ_VENV"
                                    else
                                        echo "python3-venv (with ensurepip) is required to install Azure CLI in an isolated environment."
                                        echo "Install python3-venv in the Jenkins agent image."
                                        exit 1
                                    fi
                                fi
                                "$AZ_VENV/bin/python" -m pip install --upgrade pip
                                "$AZ_VENV/bin/pip" install azure-cli
                            fi
                        fi

                        if [ -z "$AZ_BIN" ]; then
                            echo "Azure CLI executable could not be resolved after installation."
                            exit 1
                        fi

                        "$AZ_BIN" login --service-principal \
                            -u "$AZ_CLIENT_ID" \
                            -p "$AZ_CLIENT_SECRET" \
                            --tenant "$TENANT_ID"

                        "$AZ_BIN" webapp config container set \
                            --resource-group "$RESOURCE_GROUP" \
                            --name "$WEBAPP_NAME" \
                            --docker-custom-image-name "$ACR_NAME.azurecr.io/$IMAGE_NAME:$IMAGE_TAG" \
                            --docker-registry-server-url "https://$ACR_NAME.azurecr.io" \
                            --docker-registry-server-user "$ACR_USER" \
                            --docker-registry-server-password "$ACR_PASS"
                    '''
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
