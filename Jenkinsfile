pipeline {
    agent {
        docker {
            image 'python:3.11'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/PHATSAWUT-DG/Jenkins_Demopipe01.git'
            }
        }
        stage('Setup venv') {
            steps {
                sh '''
                python3 -m venv venv
                . venv/bin/activate
                pip install --upgrade pip
                pip install -r requirements.txt
                '''
            }
        }
        stage('Run Tests & Coverage') {
            steps {
                sh '''
                . venv/bin/activate
                export PYTHONPATH=.
                pytest --maxfail=1 --disable-warnings -q --cov=app --cov-report=xml
                '''
            }
        }
        // ============ NEW STAGE ============
        stage('Install SonarScanner') {
            steps {
                sh '''
                echo "Cleaning up previous files..."
                rm -f sonar-scanner.zip
                rm -rf sonar-scanner sonar-scanner-cli-7.2.0.5079-linux-x64        

                echo "Downloading SonarScanner..."
                curl -sSL -o sonar-scanner.zip https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-7.2.0.5079-linux-x64.zip?_gl=1*j7wxsl*_gcl_au*NDY1NzU2ODYzLjE3NTcyNzQ2MjA.*_ga*MjIyNzYzMTkuMTc1NzI3NDYxOQ..*_ga_9JZ0GZ5TC6*czE3NTcyNzQ2MTgkbzEkZzEkdDE3NTcyNzQ3MjkkajYwJGwwJGgw
                unzip -q -o sonar-scanner.zip
                ls -la
                mv sonar-scanner-7.2.0.5079-linux-x64 sonar-scanner
               # Adding to PATH here is fine for this step, but won't persist to the next stage.
                export PATH=$PATH:$WORKSPACE/sonar-scanner/bin
                echo "SonarScanner installed."
                '''
            }
        }
        // ===================================
        stage('SonarQube Analysis') {
            steps {
                // Use withSonarQubeEnv primarily to get the SONAR_HOST_URL based on the server name 'SonarQube Scanner'
                // configured in Jenkins (Manage Jenkins > Configure System).
                withSonarQubeEnv('SonarQube Scanner') { // Make sure this name matches your Jenkins config
                    // Explicitly bind the secret text credential to a variable
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
                            # Pass the host URL to the scanner
                            SONAR_SCANNER_OPTS="-Dsonar.host.url=$SONAR_HOST_URL"
                        else
                            echo "WARNING: SONAR_HOST_URL not provided by withSonarQubeEnv. Using default or sonar-project.properties."
                            SONAR_SCANNER_OPTS=""
                        fi

                        # Ensure sonar-scanner is in PATH
                        export PATH=$PATH:$WORKSPACE/sonar-scanner/bin

                        # Run the scanner, explicitly passing the token and potentially the host URL
                        # The token is passed via the SONAR_TOKEN environment variable, which the scanner recognizes.
                        # SONAR_TOKEN takes precedence over sonar.login property file.
                        export SONAR_TOKEN="$SCANNER_TOKEN"
                        sonar-scanner $SONAR_SCANNER_OPTS
                        '''
                    }
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                // --- NEW: Install Docker CLI inside the python container ---
                sh '''
                echo "Installing Docker CLI..."
                # Update package list
                apt-get update
                # Install prerequisites for https repositories
                apt-get install -y ca-certificates curl gnupg lsb-release
                # Add Docker's official GPG key
                mkdir -p /etc/apt/keyrings
                curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
                # Set up the repository
                echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
                # Update package list again
                apt-get update
                # Install Docker CLI (docker-ce-cli)
                apt-get install -y docker-ce-cli
                echo "Docker CLI installation complete."
                docker --version # Verify installation
                '''
                // --- END OF NEW INSTALLATION STEPS ---
                sh 'docker build -t fastapi-app:latest .'
            }
        }
        stage('Deploy Container') {
            steps {
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
        }
    }
}