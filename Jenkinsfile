pipeline {
    agent any

    environment {
        STORAGE_ACCOUNT = "chunkhoundstorage"
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
    }

    post {
        always {
            archiveArtifacts artifacts: 'dist/*', fingerprint: true
        }
    }
}
