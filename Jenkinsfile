pipeline {
    agent any

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
                sh 'echo "Workspace cleaned"'
            }
        }

        stage('Checkout') {
            steps {
            // Use the simpler `git` step which is more robust in Declarative pipelines
            git branch: 'main', url: 'https://github.com/PHATSAWUT-DG/Jenkins_Demopipe01.git'

            sh '''
            echo "=== FILES AFTER CHECKOUT ==="
            ls -la
            '''
            }
        }

        stage('Setup venv & Install Dependencies') {
            steps {
                sh '''
                echo "=== AGENT NODE VERIFICATION ==="
                echo "Workspace path: $WORKSPACE"
                echo "Contents of workspace:"
                ls -la "$WORKSPACE"
                
                if [ ! -f "$WORKSPACE/requirements.txt" ]; then
                  echo "ERROR: requirements.txt not found in workspace on agent!"
                  exit 1
                else
                  echo "SUCCESS: Found requirements.txt on agent node."
                fi
                echo "=== END AGENT NODE VERIFICATION ==="
                '''
            }
        }

        stage('Run Tests & Coverage') {
            steps {
                sh '''
                echo "Running tests inside Docker container..."

                docker run --rm \
                -v "$WORKSPACE:/workspace" \
                -w /workspace \
                python:3.11 \
                bash -c "
                    set -e
                    echo '=== CONTENTS IN CONTAINER ==='
                    ls -la /workspace
                    
                    if [ ! -f /workspace/requirements.txt ]; then
                        echo 'ERROR: requirements.txt NOT FOUND!'
                        exit 1
                    fi
                    
                    echo 'Installing dependencies...'
                    pip install --no-cache-dir -r requirements.txt
                    echo 'Dependencies installed. Running tests...'
                    export PYTHONPATH=.
                    pytest --maxfail=1 --disable-warnings -q --cov=app --cov-report=xml
                    echo 'Tests completed successfully.'
                "
                '''
            }
        }

        // ============ SONARSCANNER INSTALL STAGE ============
        // You might want to install SonarScanner on the Jenkins agent node itself
        // or download it fresh each time. Installing on the agent is more efficient.
        stage('Install SonarScanner') {
            steps {
                // This runs on the Jenkins agent node
                sh '''
                echo "Cleaning up previous SonarScanner files..."
                rm -f sonar-scanner.zip
                rm -rf sonar-scanner sonar-scanner-cli-7.2.0.5079-linux-x64

                echo "Downloading SonarScanner..."
                curl -sSL -o sonar-scanner.zip https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-7.2.0.5079-linux-x64.zip
                unzip -q -o sonar-scanner.zip
                mv sonar-scanner-7.2.0.5079-linux-x64 sonar-scanner
                echo "SonarScanner installed on Jenkins agent."
                '''
            }
        }
        // ===================================

        stage('SonarQube Analysis') {
            steps {
                // This runs on the Jenkins agent node, using the locally installed scanner
                // The scanner will analyze the code in the $WORKSPACE directory
                withSonarQubeEnv('SonarQube Scanner') { // Make sure this name matches your Jenkins config
                    withCredentials([string(credentialsId: 'sonarqube_token', variable: 'SCANNER_TOKEN')]) {
                        sh '''
                        # Debug: Confirm SCANNER_TOKEN is set from withCredentials
                        if [ -n "$SCANNER_TOKEN" ]; then
                            echo "SUCCESS: SCANNER_TOKEN (from withCredentials) is set."
                        else
                            echo "ERROR: SCANNER_TOKEN is NOT set. Check the credentialsId 'sonarqube_token'."
                            exit 1
                        fi

                        # Debug: Check if withSonarQubeEnv provided SONAR_HOST_URL
                        if [ -n "$SONAR_HOST_URL" ]; then
                            echo "INFO: SONAR_HOST_URL provided by withSonarQubeEnv: $SONAR_HOST_URL"
                            SONAR_SCANNER_OPTS="-Dsonar.host.url=$SONAR_HOST_URL"
                        else
                            echo "WARNING: SONAR_HOST_URL not provided by withSonarQubeEnv. Using default or sonar-project.properties."
                            SONAR_SCANNER_OPTS=""
                        fi

                        # Ensure sonar-scanner is in PATH (adjust path if needed)
                        export PATH=$PATH:$WORKSPACE/sonar-scanner/bin

                        # Run the scanner, explicitly passing the token
                        export SONAR_TOKEN="$SCANNER_TOKEN"
                        sonar-scanner $SONAR_SCANNER_OPTS
                        '''
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                // This runs directly on the Jenkins agent node, where Docker CLI should be available
                sh 'docker build -t fastapi-app:latest .'
            }
        }

        stage('Deploy Container') {
            steps {
                 // This also runs directly on the Jenkins agent node
                sh '''
                docker stop fastapi_app || true
                docker rm fastapi_app || true
                docker run -d -p 8000:8000 --name fastapi_app fastapi-app:latest
                '''
            }
        }
    }
    post {
        always {
            echo "Pipeline finished"
             // Optional: Cleanup downloaded scanner files
            // sh 'rm -rf sonar-scanner sonar-scanner.zip || true'
        }
    }
}