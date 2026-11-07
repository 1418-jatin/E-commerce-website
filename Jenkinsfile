pipeline {
    // Runs on the EC2 instance registered in Jenkins as a node with label "linux-agent"
    agent { label 'linux-agent' }

    environment {
        // Username/password credential (ID: dockerhub-cred)
        // Exposes DOCKERHUB_CRED_USR and DOCKERHUB_CRED_PSW
        DOCKERHUB_CRED = credentials('dockerhub-cred')

        APP_NAME      = 'stylo-shop'
        IMAGE_REPO    = '19901418/prt-oct-25th'
        IMAGE_TAG     = "${BUILD_NUMBER}"

        // Private IP of the k8s-node EC2 instance (same VPC as ci-agent)
        K8S_NODE_IP   = '172.31.45.11'
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
                // k8s-node-ssh = "SSH Username with private key" credential (key from ~/.ssh/jenkins_k8s)
                sshagent(credentials: ['k8s-node-ssh']) {
                    sh '''
                        # Inject the freshly built image tag into the deployment manifest
                        sed -i "s|IMAGE_PLACEHOLDER|${IMAGE_REPO}:${IMAGE_TAG}|g" k8s/deployment.yml

                        SSH_OPTS="-o StrictHostKeyChecking=no"
                        REMOTE="${K8S_NODE_USER}@${K8S_NODE_IP}"
                        REMOTE_DIR="~/${APP_NAME}-k8s"

                        # Copy both manifests to the Kubernetes node
                        ssh $SSH_OPTS $REMOTE "mkdir -p ${REMOTE_DIR}"
                        scp $SSH_OPTS k8s/deployment.yml k8s/service.yml $REMOTE:${REMOTE_DIR}/

                        # Apply deployment + service, then wait for the rollout
                        ssh $SSH_OPTS $REMOTE "
                            export PATH=\\$PATH:/snap/bin:/usr/local/bin &&
                            kubectl apply -f ${REMOTE_DIR}/deployment.yml &&
                            kubectl apply -f ${REMOTE_DIR}/service.yml &&
                            kubectl rollout status deployment/${APP_NAME} --timeout=120s &&
                            kubectl get pods,svc -l app=${APP_NAME}
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
            echo "Deployed ${IMAGE_REPO}:${IMAGE_TAG} to the Kubernetes node."
        }
        failure {
            echo 'Pipeline failed. Check the stage logs above.'
        }
    }
}
