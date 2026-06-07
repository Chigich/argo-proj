// =============================================================
// Jenkinsfile — Full CI/CD Pipeline
// Stages: Checkout → SonarQube → Build → ECR Push → Deploy
// =============================================================

pipeline {
    agent any

    environment {
        // AWS settings — configure these in Jenkins credentials
        AWS_REGION        = 'us-east-1'
        AWS_ACCOUNT_ID    = credentials('aws-account-id')
        ECR_REPO_NAME     = 'my-app'
        ECR_REGISTRY      = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_NAME        = "${ECR_REGISTRY}/${ECR_REPO_NAME}"
        IMAGE_TAG         = "${BUILD_NUMBER}-${GIT_COMMIT[0..6]}"

        // SonarQube settings
        SONAR_PROJECT_KEY = 'my-app'

        // GitOps manifest repo
        MANIFEST_REPO     = 'https://github.com/YOUR_ORG/my-app-manifests.git'
        MANIFEST_REPO_CRED = 'github-credentials'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        timestamps()
    }

    stages {

        // ─────────────────────────────────────────────────────────────
        // STAGE 1: Checkout
        // ─────────────────────────────────────────────────────────────
        stage('Checkout') {
            steps {
                echo "=== Stage 1: Checking out source code ==="
                checkout scm
                sh 'git log --oneline -5'
                sh 'ls -la'
            }
        }

        // ─────────────────────────────────────────────────────────────
        // STAGE 2: SonarQube Analysis
        // ─────────────────────────────────────────────────────────────
        stage('SonarQube Analysis') {
            steps {
                echo "=== Stage 2: Running SonarQube analysis ==="

                // Install deps and run tests with coverage for SonarQube
                sh '''
                    pip install -r requirements.txt
                    python -m pytest tests/ \
                        --cov=src \
                        --cov-report=xml:coverage.xml \
                        --cov-report=term \
                        -v || true
                '''

                // Run SonarQube scan
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        sonar-scanner \
                          -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                          -Dsonar.sources=src \
                          -Dsonar.python.coverage.reportPaths=coverage.xml
                    '''
                }

                // Wait for Quality Gate result
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ─────────────────────────────────────────────────────────────
        // STAGE 3: Build Docker Image
        // ─────────────────────────────────────────────────────────────
        stage('Build Docker Image') {
            steps {
                echo "=== Stage 3: Building Docker image ==="
                sh '''
                    docker build \
                      --build-arg APP_VERSION=${BUILD_NUMBER} \
                      -t ${IMAGE_NAME}:${IMAGE_TAG} \
                      -t ${IMAGE_NAME}:latest \
                      .
                '''
                sh "docker images | grep ${ECR_REPO_NAME}"
            }
        }

        // ─────────────────────────────────────────────────────────────
        // STAGE 4: Push to AWS ECR
        // ─────────────────────────────────────────────────────────────
        stage('Push to ECR') {
            steps {
                echo "=== Stage 4: Pushing image to AWS ECR ==="
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-credentials',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh '''
                        # Authenticate Docker to ECR
                        aws ecr get-login-password --region ${AWS_REGION} | \
                          docker login --username AWS --password-stdin ${ECR_REGISTRY}

                        # Create ECR repo if it doesn't exist
                        aws ecr describe-repositories \
                          --repository-names ${ECR_REPO_NAME} \
                          --region ${AWS_REGION} || \
                        aws ecr create-repository \
                          --repository-name ${ECR_REPO_NAME} \
                          --region ${AWS_REGION}

                        # Push both tags
                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${IMAGE_NAME}:latest

                        echo "Image pushed: ${IMAGE_NAME}:${IMAGE_TAG}"
                    '''
                }
            }
        }

        // ─────────────────────────────────────────────────────────────
        // STAGE 5: Update Kubernetes Manifest (GitOps trigger)
        // ArgoCD watches the manifest repo and auto-deploys on changes
        // ─────────────────────────────────────────────────────────────
        stage('Update K8s Manifest') {
            steps {
                echo "=== Stage 5: Updating Kubernetes manifest in GitOps repo ==="
                withCredentials([usernamePassword(
                    credentialsId: env.MANIFEST_REPO_CRED,
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_TOKEN'
                )]) {
                    sh '''
                        # Clone the manifest repo
                        git clone https://${GIT_USER}:${GIT_TOKEN}@github.com/YOUR_ORG/my-app-manifests.git manifest-repo
                        cd manifest-repo

                        # Update the image tag in deployment.yaml using sed
                        sed -i "s|image: .*|image: ${IMAGE_NAME}:${IMAGE_TAG}|g" deployment.yaml

                        # Commit and push — ArgoCD will detect this and sync
                        git config user.email "jenkins@ci.local"
                        git config user.name "Jenkins CI"
                        git add deployment.yaml
                        git commit -m "ci: update image to ${IMAGE_TAG} [build ${BUILD_NUMBER}]"
                        git push origin main

                        echo "Manifest updated. ArgoCD will sync automatically."
                    '''
                }
            }
        }
    }

    // ─────────────────────────────────────────────────────────────
    // Post-build actions
    // ─────────────────────────────────────────────────────────────
    post {
        always {
            // Clean up Docker images to save disk space
            sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
            sh "docker rmi ${IMAGE_NAME}:latest || true"
            cleanWs()
        }
        success {
            echo "Pipeline succeeded! Image ${IMAGE_NAME}:${IMAGE_TAG} deployed via ArgoCD."
        }
        failure {
            echo "Pipeline failed at stage: ${env.STAGE_NAME}"
            // Add email/Slack notification here
        }
    }
}
