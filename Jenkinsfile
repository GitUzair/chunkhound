pipeline {
    agent any

    environment {
        STORAGE_ACCOUNT = "chunkhoundstorage"
        CONTAINER_NAME = "chunkhound-artifact"
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
                    uv run pytest || true
                '''
            }
        }

        stage('Build Package') {
            steps {
                sh '''
                    uv build
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
