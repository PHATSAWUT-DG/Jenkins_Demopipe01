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
                withSonarQubeEnv('SonarQube Scanner') {
                    sh '''
                    # Debug: Check if SONAR_TOKEN is set by withSonarQubeEnv
                    if [ -n "$SONAR_TOKEN" ]; then
                        echo "SUCCESS: SONAR_TOKEN is set."
                    else
                        echo "ERROR: SONAR_TOKEN is NOT set. Check Jenkins SonarQube server configuration and credential association."
                        exit 1
                    fi

                    # Ensure sonar-scanner is in PATH or use full path
                    # Option 1: Re-add to PATH for this shell session
                    export PATH=$PATH:$WORKSPACE/sonar-scanner/bin
                    sonar-scanner

                    # Option 2: Use full path (alternative to modifying PATH)
                    # $WORKSPACE/sonar-scanner/bin/sonar-scanner
                    '''
                }
            }
        }
        stage('Build Docker Image') {
            steps {
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