pipeline {
agent any

```
environment {
    // AWS settings
    AWS_REGION         = 'us-east-1'
    AWS_ACCOUNT_ID     = credentials('aws-account-id')
    ECR_REPO_NAME      = 'my-application'
    ECR_REGISTRY       = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
    IMAGE_NAME         = "${ECR_REGISTRY}/${ECR_REPO_NAME}"
    IMAGE_TAG          = "${BUILD_NUMBER}"

    // SonarQube settings
    SONAR_PROJECT_KEY  = 'my-application'

    // GitOps manifest repo
    MANIFEST_REPO      =  'MANIFEST_REPO' = 'https://github.com/Chigich/my-app-manifests.git'
    MANIFEST_REPO_CRED = 'git-credentials'
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
            echo "=== Stage 1: Checking out source code ==="
            checkout scm
            sh 'git log --oneline -5'
            sh 'ls -la'
        }
    }

    stage('SonarQube Analysis') {
        steps {
            echo "=== Stage 2: Running SonarQube analysis ==="

            sh '''
                pip install -r requirements.txt

            '''

            withSonarQubeEnv('SonarQube') {
                sh '''
                    sonar-scanner \
                      -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                      -Dsonar.sources=argocd \
                '''
            }

            timeout(time: 5, unit: 'MINUTES') {
                waitForQualityGate abortPipeline: true
            }
        }
    }

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

    stage('Push to ECR') {
        steps {
            echo "=== Stage 4: Pushing image to AWS ECR ==="

            withCredentials([
                string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'),
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

                    echo "Image pushed: ${IMAGE_NAME}:${IMAGE_TAG}"
                '''
            }
        }
    }

    stage('Update K8s Manifest') {
        steps {
            echo "=== Stage 5: Updating Kubernetes manifest in GitOps repo ==="

            withCredentials([
                usernamePassword(
                    credentialsId: env.MANIFEST_REPO_CRED,
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_TOKEN'
                )
            ]) {

                sh '''
                    git clone https://${GIT_USER}:${GIT_TOKEN}@github.com/Chigich/my-app-manifests.git manifest-repo

                    cd manifest-repo

                    sed -i "s|image: .*|image: ${IMAGE_NAME}:${IMAGE_TAG}|g" deployment.yaml

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

post {
    always {
        sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
        sh "docker rmi ${IMAGE_NAME}:latest || true"
        cleanWs()
    }

    success {
        echo "Pipeline succeeded! Image ${IMAGE_NAME}:${IMAGE_TAG} deployed via ArgoCD."
    }

    failure {
        echo "Pipeline failed at stage: ${env.STAGE_NAME}"
    }
}
```

}
