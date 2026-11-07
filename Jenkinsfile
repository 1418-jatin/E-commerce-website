pipeline {
    // Runs on the EC2 instance registered as a Jenkins node with label "ci-agent"
    agent { label 'ci-agent' }

    environment {
        // Username/password credential (ID: dockerhub-cred)
        // Exposes DOCKERHUB_CRED_USR and DOCKERHUB_CRED_PSW
        DOCKERHUB_CRED = credentials('dockerhub-cred')

        APP_NAME     = 'stylo-shop'
        IMAGE_REPO   = "${DOCKERHUB_CRED_USR}/${APP_NAME}"
        IMAGE_TAG    = "${BUILD_NUMBER}"

        // >>> CHANGE THIS: private/public IP of your Kubernetes node EC2 instance
        K8S_NODE_IP  = 'REPLACE_WITH_K8S_NODE_IP'
        K8S_NODE_USER = 'ubuntu'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${IMAGE_REPO}:${IMAGE_TAG} -t ${IMAGE_REPO}:latest .
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                sh '''
                    echo "${DOCKERHUB_CRED_PSW}" | docker login -u "${DOCKERHUB_CRED_USR}" --password-stdin
                    docker push ${IMAGE_REPO}:${IMAGE_TAG}
                    docker push ${IMAGE_REPO}:latest
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                // ubuntu-cred = "SSH Username with private key" credential
                sshagent(credentials: ['ubuntu-cred']) {
                    sh '''
                        # Inject the freshly built image tag into the manifest
                        sed -i "s|IMAGE_PLACEHOLDER|${IMAGE_REPO}:${IMAGE_TAG}|g" k8s/deployment.yaml

                        SSH_OPTS="-o StrictHostKeyChecking=no"

                        ssh $SSH_OPTS ${K8S_NODE_USER}@${K8S_NODE_IP} "mkdir -p ~/${APP_NAME}-k8s"
                        scp $SSH_OPTS k8s/*.yaml ${K8S_NODE_USER}@${K8S_NODE_IP}:~/${APP_NAME}-k8s/

                        ssh $SSH_OPTS ${K8S_NODE_USER}@${K8S_NODE_IP} "
                            kubectl apply -f ~/${APP_NAME}-k8s/ &&
                            kubectl rollout status deployment/${APP_NAME} --timeout=120s
                        "
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
            sh 'docker image prune -f || true'
        }
        success {
            echo "Deployed ${IMAGE_REPO}:${IMAGE_TAG} -> http://${K8S_NODE_IP}:30080"
        }
        failure {
            echo 'Pipeline failed. Check the stage logs above.'
        }
    }
}
