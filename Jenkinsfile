pipeline {
    agent { label 'docker-node' }

    triggers {
        githubPush() // Make sure GitHub webhooks are configured
    }

    environment {
        AWS_REGION = 'ap-south-1'
        ECR_REPO = '699475940685.dkr.ecr.ap-south-1.amazonaws.com/docker/calci'
        IMAGE_TAG = "calci-${BUILD_NUMBER}"
        DEPLOY_HOST = "ec2-user@172.31.3.248"
        GIT_REPO = 'https://github.com/Raveendra-hm/docker-sample-java-webapp.git'
        KUBE_CONFIG_CREDENTIAL_ID = 'kubeconfig-remote-cluster'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

    stage('Determine Namespace') {
        when {
            expression {
                return ["dev", "test", "uat"].contains(env.BRANCH_NAME)
            }
        }
        environment {
            NAMESPACE = "${BRANCH_NAME}"
        }
        steps {
            echo "Branch: ${BRANCH_NAME} ➝ Namespace: ${NAMESPACE}"
        }
}


        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${ECR_REPO}:${IMAGE_TAG} ."
                }
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}
                        docker push ${ECR_REPO}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: "${KUBE_CONFIG_CREDENTIAL_ID}", variable: 'KUBECONFIG')]) {
                    sh """
                        export KUBECONFIG=$KUBECONFIG
                        echo "Deploying to Kubernetes namespace: ${NAMESPACE}"

                        # Replace placeholder with the new image
                        sed -i "s|IMAGE_PLACEHOLDER|${ECR_REPO}:${IMAGE_TAG}|g" k8s/deployment.yaml

                        # Apply all manifests
                        kubectl apply -n ${NAMESPACE} -f k8s/
                    """
                }
            }
        }
    }
}