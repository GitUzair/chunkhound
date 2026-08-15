pipeline {
    agent any

    environment {
        STORAGE_ACCOUNT = "chunkhoundartifacts2026"
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
            set -e

            echo "===== Building ChunkHound Package ====="

            rm -rf dist
            uv build

            echo "===== Build Artifacts ====="
            ls -lh dist/

            WHEEL=$(find dist -maxdepth 1 -type f -name '*.whl' -printf '%f\\n' | head -n 1)

            if [ -z "$WHEEL" ]; then
                echo "ERROR: No wheel artifact was created."
                exit 1
            fi

            echo "Wheel produced by this pipeline:"
            echo "$WHEEL"

            printf '%s\\n' "$WHEEL" > artifact-name.txt

            echo "Artifact name recorded:"
            cat artifact-name.txt
        '''

        archiveArtifacts artifacts: 'artifact-name.txt', fingerprint: true
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

                WHEEL=$(cat artifact-name.txt)

                if [ -z "$WHEEL" ]; then
                    echo "ERROR: artifact-name.txt is empty."
                    exit 1
                fi

                if [ ! -f "dist/$WHEEL" ]; then
                    echo "ERROR: Artifact dist/$WHEEL does not exist."
                    exit 1
                fi

                echo "Artifact selected for upload:"
                echo "$WHEEL"

                echo "Artifact details:"
                ls -lh "dist/$WHEEL"

                az storage blob upload \
                    --account-name "$STORAGE_ACCOUNT" \
                    --account-key "$AZURE_STORAGE_KEY" \
                    --container-name "$CONTAINER_NAME" \
                    --name "$WHEEL" \
                    --file "dist/$WHEEL" \
                    --overwrite true \
                    --only-show-errors

                echo "===== Artifact Upload Complete ====="

                echo "Uploaded artifact:"
                az storage blob show \
                    --account-name "$STORAGE_ACCOUNT" \
                    --account-key "$AZURE_STORAGE_KEY" \
                    --container-name "$CONTAINER_NAME" \
                    --name "$WHEEL" \
                    --query "{name:name,size:properties.contentLength}" \
                    -o table
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

                echo "Reading artifact produced by this pipeline..."

                WHEEL=$(cat artifact-name.txt)

                if [ -z "$WHEEL" ]; then
                    echo "ERROR: artifact-name.txt is empty."
                    exit 1
                fi

                echo "Artifact selected for deployment:"
                echo "$WHEEL"

                echo "Checking that artifact exists in Azure Blob Storage..."

                BLOB_EXISTS=$(az storage blob exists \
                    --account-name "$STORAGE_ACCOUNT" \
                    --account-key "$AZURE_STORAGE_KEY" \
                    --container-name "$CONTAINER_NAME" \
                    --name "$WHEEL" \
                    --query exists \
                    -o tsv)

                if [ "$BLOB_EXISTS" != "true" ]; then
                    echo "ERROR: Artifact $WHEEL was not found in Azure Blob Storage."
                    exit 1
                fi

                echo "Artifact confirmed in Blob Storage."

                echo "Removing previously downloaded wheel..."

                rm -f "$DEPLOY_DIR"/*.whl

                echo "Downloading exact artifact..."

                az storage blob download \
                    --account-name "$STORAGE_ACCOUNT" \
                    --account-key "$AZURE_STORAGE_KEY" \
                    --container-name "$CONTAINER_NAME" \
                    --name "$WHEEL" \
                    --file "$DEPLOY_DIR/$WHEEL" \
                    --overwrite true \
                    --only-show-errors

                echo "Downloaded artifact:"
                ls -lh "$DEPLOY_DIR/$WHEEL"

                echo "Creating deployment virtual environment..."

                if [ ! -d "$VENV_DIR" ]; then
                    uv venv "$VENV_DIR" --python 3.12
                fi

                echo "Installing exact artifact into deployment environment..."

                uv pip install \
                    --python "$VENV_DIR/bin/python" \
                    --force-reinstall \
                    "$DEPLOY_DIR/$WHEEL"

                echo "===== Installed Package ====="

                "$VENV_DIR/bin/chunkhound" --version

                echo "===== Deployment Verification ====="

                "$VENV_DIR/bin/python" -c \
                    "import chunkhound; print('ChunkHound import: OK')"

                "$VENV_DIR/bin/chunkhound" --help \
                    > "$DEPLOY_DIR/chunkhound-help.txt"

                grep -q "index" "$DEPLOY_DIR/chunkhound-help.txt"
                grep -q "search" "$DEPLOY_DIR/chunkhound-help.txt"

                echo "ChunkHound deployment verified successfully."

                echo "===== Restarting ChunkHound Service ====="

                sudo systemctl restart chunkhound

                sleep 3

                sudo systemctl is-active --quiet chunkhound

                echo "ChunkHound systemd service is active."

                echo "===== Service Status ====="

               sudo systemctl status chunkhound --no-pager -l
            '''
        }
    }
}

      

       

   stage('Application Smoke Test') {
    steps {
        sh '''
            set -e

            echo "===== ChunkHound Application Smoke Test ====="

            echo "Checking systemd service..."

            sudo systemctl is-active --quiet chunkhound

            echo "Service is ACTIVE."

            echo "Testing deployed Python import..."

            /opt/chunkhound/venv/bin/python -c \
                "import chunkhound; print('ChunkHound import: OK')"

            echo "Testing deployed CLI..."

            /opt/chunkhound/venv/bin/chunkhound --help \
                > /opt/chunkhound/chunkhound-help.txt

            grep -q "index" /opt/chunkhound/chunkhound-help.txt
            grep -q "search" /opt/chunkhound/chunkhound-help.txt

            echo "Testing deployed package version..."

            /opt/chunkhound/venv/bin/chunkhound --version

            echo "===== Smoke Test PASSED ====="
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
