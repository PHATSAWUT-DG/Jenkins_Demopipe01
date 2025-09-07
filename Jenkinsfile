pipeline {
    // Run the pipeline steps directly on the Jenkins agent node (must have Docker installed)
    agent any

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout') {
            steps {
                // This will run directly on the Jenkins agent node
                git branch: 'main', url: 'https://github.com/PHATSAWUT-DG/Jenkins_Demopipe01.git'
            }
        }

        stage('Setup venv & Install Dependencies') {
            steps {
                // --- VERIFICATION STEP ON AGENT NODE ---
                sh '''
                echo "=== AGENT NODE VERIFICATION ==="
                echo "Current user: $(id -u -n):$(id -g -n)"
                echo "Current user ID: $(id -u):$(id -g)"
                echo "Workspace path: $WORKSPACE"
                echo "Contents of workspace BEFORE marker file:"
                ls -la "$WORKSPACE"

                if [ ! -f "$WORKSPACE/requirements.txt" ]; then
                  echo "ERROR: requirements.txt not found in workspace on agent!"
                  exit 1
                else
                  echo "SUCCESS: Found requirements.txt on agent node."
                fi

                # Create a unique marker file to verify mounting
                echo "This file was created on the host at $(date)" > "$WORKSPACE/DEBUG_HOST_MARKER.txt"
                echo "Created DEBUG_HOST_MARKER.txt"
                ls -la "$WORKSPACE/DEBUG_HOST_MARKER.txt"
                echo "=== END AGENT NODE VERIFICATION ==="
                '''
                // --- END OF AGENT VERIFICATION ---

                // --- STEP 1: Simple Docker Volume Mount Test with Alpine ---
                sh '''
                echo "=== STEP 1: Alpine Volume Mount Test ==="
                echo "Running Alpine container to list mounted workspace contents:"
                docker run --rm \
                  -v "$WORKSPACE:/workspace" \
                  -w /workspace \
                  alpine:latest \
                  sh -c "echo Contents of /workspace; ls -la; echo Contents of / including /workspace; ls -la /"
                echo "=== END STEP 1 ==="
                '''
                // --- END OF ALPINE TEST ---

                // --- STEP 2: Python Container Mount Test ---
                sh '''
                echo "=== STEP 2: Python Container Mount Test ==="
                echo "Running Python container to list mounted workspace contents:"
                docker run --rm \
                  -v "$WORKSPACE:/workspace" \
                  -w /workspace \
                  python:3.11 \
                  bash -c "echo Contents of /workspace; ls -la; echo Contents of / including /workspace; ls -la /"
                echo "=== END STEP 2 ==="
                '''
                // --- END OF PYTHON TEST ---
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