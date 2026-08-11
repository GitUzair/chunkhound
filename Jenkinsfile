pipeline {
    agent any

    environment {
        STORAGE_ACCOUNT = "chunkhoundstorageacc"
        CONTAINER_NAME = "chunkhound-artifact"
        SONAR_SCANNER_HOME = tool 'SonarScanner'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Environment Check') {
            steps {
                sh '''
                    echo "Python:"
                    python3 --version

                    echo "UV:"
                    uv --version

                    echo "Git:"
                    git --version
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    uv sync --dev
                '''
            }
        }

        stage('Code Quality') {
            steps {
                sh '''
                    echo "Running Ruff..."
                    uv run ruff check . || true

                    echo "Running MyPy..."
                    uv run mypy chunkhound || true

                    echo "Checking Tree-sitter..."
                    uv run python -c "import tree_sitter; print('Tree-sitter OK')"
                '''
            }
        }

        stage('Unit Tests') {
            steps {
                sh '''
                    echo "Running Unit Tests..."
                    uv run pytest || true
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        echo "Running SonarQube Scanner..."
                        "${SONAR_SCANNER_HOME}/bin/sonar-scanner"
                    '''
                }
            }
        }

        stage('Quality Gate') {
    steps {
        timeout(time: 5, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: false
            }
        }
    }  

        stage('Build Package') {
            steps {
                sh '''
                    uv build
                '''
            }
        }


    stage('Upload Artifact to Blob Storage') {
    steps {
        withCredentials([
            string(credentialsId: 'azure-storage-key', variable: 'AZURE_STORAGE_KEY')
        ]) {
            sh '''
                set -e

                echo "===== Uploading Artifact to Azure Blob Storage ====="

                echo "Storage Account: ${STORAGE_ACCOUNT}"
                echo "Container: ${CONTAINER_NAME}"

                echo "Artifacts to upload:"
                ls -lh dist/

                for artifact in dist/*; do
                    echo "Uploading $(basename "$artifact")..."

                    az storage blob upload \
                        --account-name "$STORAGE_ACCOUNT" \
                        --account-key "$AZURE_STORAGE_KEY" \
                        --container-name "$CONTAINER_NAME" \
                        --name "$(basename "$artifact")" \
                        --file "$artifact" \
                        --overwrite true \
                        --only-show-errors
                done

                echo "===== Artifact Upload Complete ====="
            '''
        }
    }
}

stage('Deploy to VM') {
    steps {
        withCredentials([
            string(credentialsId: 'azure-storage-key', variable: 'AZURE_STORAGE_KEY')
        ]) {
            sh '''
                set -e

                echo "===== ChunkHound Deployment ====="

                DEPLOY_DIR="/opt/chunkhound"
                VENV_DIR="$DEPLOY_DIR/venv"

                echo "Creating deployment directory..."

                sudo mkdir -p "$DEPLOY_DIR"
                sudo chown -R jenkins:jenkins "$DEPLOY_DIR"

                echo "Finding wheel artifact in Azure Blob Storage..."

                WHEEL=$(az storage blob list \
                    --account-name "$STORAGE_ACCOUNT" \
                    --account-key "$AZURE_STORAGE_KEY" \
                    --container-name "$CONTAINER_NAME" \
                    --query "[?ends_with(name, '.whl')].name | [0]" \
                    -o tsv)

                if [ -z "$WHEEL" ]; then
                    echo "ERROR: No wheel artifact found in Azure Blob Storage."
                    exit 1
                fi

                echo "Selected artifact: $WHEEL"

                echo "Downloading artifact..."

                rm -f "$DEPLOY_DIR"/*.whl

                az storage blob download \
                    --account-name "$STORAGE_ACCOUNT" \
                    --account-key "$AZURE_STORAGE_KEY" \
                    --container-name "$CONTAINER_NAME" \
                    --name "$WHEEL" \
                    --file "$DEPLOY_DIR/$(basename "$WHEEL")" \
                    --overwrite true \
                    --only-show-errors

                echo "Downloaded artifact:"
                ls -lh "$DEPLOY_DIR"/*.whl

                echo "Creating deployment virtual environment..."

                if [ ! -d "$VENV_DIR" ]; then
                    uv venv "$VENV_DIR" --python 3.12
                fi

                echo "Installing ChunkHound artifact..."

                uv pip install \
                    --python "$VENV_DIR/bin/python" \
                    --force-reinstall \
                    "$DEPLOY_DIR/$(basename "$WHEEL")"

                echo "===== Deployment Verification ====="

                "$VENV_DIR/bin/python" -c "import chunkhound; print('ChunkHound package installed: OK')"

                "$VENV_DIR/bin/chunkhound" --version

                echo "ChunkHound deployment completed successfully."
 
                echo "===== Starting ChunkHound systemd service ====="

                  sudo systemctl daemon-reload
                  sudo systemctl enable chunkhound
                  sudo systemctl restart chunkhound

               echo "===== Verifying ChunkHound systemd service ====="

                  sudo systemctl is-active --quiet chunkhound

               echo "ChunkHound systemd service is active."
                  sudo systemctl status chunkhound --no-pager -l
            '''
        }
    }
}

    stage('Application Smoke Test') {
    steps {
        sh '''
            set -e

            echo "===== ChunkHound Deployment Smoke Test ====="

            DEPLOY_DIR="/opt/chunkhound"
            VENV_DIR="$DEPLOY_DIR/venv"

            echo "Testing deployed Python import..."

            "$VENV_DIR/bin/python" -c \
                "import chunkhound; print('ChunkHound import: OK')"

            echo "Testing deployed CLI..."

            "$VENV_DIR/bin/chunkhound" --help > "$DEPLOY_DIR/chunkhound-help.txt"

            grep -q "index" "$DEPLOY_DIR/chunkhound-help.txt"
            grep -q "search" "$DEPLOY_DIR/chunkhound-help.txt"

            echo "Testing deployed package version..."

            "$VENV_DIR/bin/chunkhound" --version

            echo "===== Deployment Smoke Test PASSED ====="
        '''
      }
  }
}

    post {
        always {
            archiveArtifacts artifacts: 'dist/*', fingerprint: true
        }
    }
}
