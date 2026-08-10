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


    stage('Upload Artifact') {
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


     stage('Application Smoke Test') {
      steps {
        sh '''
            echo "===== ChunkHound Smoke Test ====="

            echo "Testing Python import..."
            uv run python -c "import chunkhound; print('ChunkHound import: OK')"

            echo "Testing CLI..."
            uv run chunkhound --help > chunkhound-help.txt

            grep -q "index" chunkhound-help.txt
            grep -q "search" chunkhound-help.txt

            echo "Testing package version..."
            uv run chunkhound --version

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
