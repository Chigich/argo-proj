pipeline {
    agent any

    environment {
        AWS_REGION         = 'us-east-1'
        AWS_ACCOUNT_ID     = credentials('aws-account-id')

        ECR_REPO_NAME      = 'my-application'
        ECR_REGISTRY       = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_NAME         = "${ECR_REGISTRY}/${ECR_REPO_NAME}"
        IMAGE_TAG          = "${BUILD_NUMBER}"

        SONAR_PROJECT_KEY  = 'my-application'

        MANIFEST_REPO      = 'https://github.com/Chigich/GitOps-manifests.git'
        MANIFEST_REPO_CRED = 'git-credentials'

        // Ensure pip-installed tools AND sonar-scanner are on PATH
        PATH = "/var/jenkins_home/.local/bin:/var/jenkins_home/sonar-scanner/bin:${env.PATH}"
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        timestamps()
    }

    stages {

        stage('Checkout') {
            steps {
                echo "=== Stage 1: Checkout ==="
                checkout scm
                sh '''
                    pwd
                    ls -la
                    git log --oneline -5
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo "=== Stage 2: SonarQube Analysis ==="

                // Install Python dependencies
                sh '''
                    pip3 install -r requirements.txt --break-system-packages
                '''

                // Run pytest with coverage BEFORE sonar-scanner
                sh '''
                    cd argocd
                    pytest --cov=. --cov-report=xml:../coverage.xml --cov-report=term || true
                    cd ..
                    echo ">>> coverage.xml generated:"
                    ls -lh coverage.xml || echo "WARNING: coverage.xml not found"
                '''

                // Auto-install sonar-scanner if not already present on the agent
                sh '''
                    SONAR_SCANNER_VERSION="6.2.1.4610"
                    SONAR_SCANNER_HOME="/var/jenkins_home/sonar-scanner"

                    if [ ! -f "${SONAR_SCANNER_HOME}/bin/sonar-scanner" ]; then
                        echo ">>> sonar-scanner not found. Downloading..."
                        curl -sSLo /tmp/sonar-scanner.zip \
                            "https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-${SONAR_SCANNER_VERSION}-linux-x64.zip"
                        unzip -q /tmp/sonar-scanner.zip -d /tmp/sonar-scanner-extracted
                        mv /tmp/sonar-scanner-extracted/sonar-scanner-${SONAR_SCANNER_VERSION}-linux-x64 ${SONAR_SCANNER_HOME}
                        rm -f /tmp/sonar-scanner.zip
                        echo ">>> sonar-scanner installed successfully."
                    else
                        echo ">>> sonar-scanner already installed. Skipping download."
                    fi

                    echo ">>> sonar-scanner version:"
                    sonar-scanner --version
                '''

                // Run SonarQube scan (now with coverage report)
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        sonar-scanner \
                          -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                          -Dsonar.sources=argocd \
                          -Dsonar.python.version=3 \
                          -Dsonar.python.coverage.reportPaths=coverage.xml
                    '''
                }

                // Wait for Quality Gate result — abortPipeline: false means
                // the pipeline continues even if the gate is ERROR (warns only).
                // Change to: abortPipeline: true  once your code is clean.
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "=== Stage 3: Build Docker Image ==="
                sh '''
                    docker build \
                      -t ${IMAGE_NAME}:${IMAGE_TAG} \
                      -t ${IMAGE_NAME}:latest \
                      .
                '''
                sh '''
                    docker images | grep ${ECR_REPO_NAME}
                '''
            }
        }

        stage('Push to ECR') {
            steps {
                echo "=== Stage 4: Push to ECR ==="
                withCredentials([
                    string(credentialsId: 'aws-access-key-id',     variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                        export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                        export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY

                        aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login --username AWS --password-stdin ${ECR_REGISTRY}

                        aws ecr describe-repositories \
                          --repository-names ${ECR_REPO_NAME} \
                          --region ${AWS_REGION} || \
                        aws ecr create-repository \
                          --repository-name ${ECR_REPO_NAME} \
                          --region ${AWS_REGION}

                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${IMAGE_NAME}:latest

                        echo "Pushed image: ${IMAGE_NAME}:${IMAGE_TAG}"
                    '''
                }
            }
        }

        stage('Update GitOps Manifest') {
            steps {
                echo "=== Stage 5: Update GitOps Repository ==="
                withCredentials([
                    usernamePassword(
                        credentialsId: 'git-credentials',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN'
                    )
                ]) {
                    sh '''
                        rm -rf manifest-repo

                        git clone https://${GIT_USER}:${GIT_TOKEN}@github.com/Chigich/GitOps-manifests.git manifest-repo

                        cd manifest-repo

                        sed -i "s|image: .*|image: ${IMAGE_NAME}:${IMAGE_TAG}|g" my-app-manifests/deployment.yaml

                        git config user.email "jenkins@local"
                        git config user.name "Jenkins"

                        git add my-app-manifests/deployment.yaml

                        git commit -m "Update image to ${IMAGE_TAG}" || true

                        git push origin main
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker image prune -f || true'
            sh 'docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true'
            sh 'docker rmi ${IMAGE_NAME}:latest || true'
            cleanWs()
        }
        success {
            echo "Pipeline completed successfully."
        }
        failure {
            echo "Pipeline failed."
        }
    }
}
