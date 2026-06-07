pipeline {
agent any

```
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

            sh '''
                pip3 install -r requirements.txt
            '''

            withSonarQubeEnv('SonarQube') {
                sh '''
                    sonar-scanner \
                      -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                      -Dsonar.sources=argocd
                '''
            }

            timeout(time: 5, unit: 'MINUTES') {
                waitForQualityGate abortPipeline: true
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

        sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
        sh "docker rmi ${IMAGE_NAME}:latest || true"

        cleanWs()
    }

    success {
        echo "Pipeline completed successfully."
    }

    failure {
        echo "Pipeline failed."
    }
}
```

}
