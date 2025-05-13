pipeline { 
    agent { label 'docker-node' }

    triggers {
        githubPush() // Trigger via GitHub webhook
    }

    environment {
        AWS_REGION = 'ap-south-1'
        ECR_REPO = '699475940685.dkr.ecr.ap-south-1.amazonaws.com/docker/calci'
        IMAGE_TAG = "calci-${BUILD_NUMBER}"
        GIT_REPO = 'https://github.com/Raveendra-hm/docker-sample-java-webapp.git'
        KUBE_CONFIG_CREDENTIAL_ID = 'kubeconfig-remote-cluster'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate Branch') {
            when {
                not {
                    expression {
                        return ["dev", "test", "uat"].contains(env.BRANCH_NAME)
                    }
                }
            }
            steps {
                script {
                    error " Branch '${env.BRANCH_NAME}' is not mapped to a Kubernetes namespace."
                }
            }
        }

        stage('Determine Namespace') {
            when {
                expression {
                    return ["dev", "test", "uat"].contains(env.BRANCH_NAME)
                }
            }
            steps {
                script {
                    env.NAMESPACE = env.BRANCH_NAME
                    echo "Branch '${env.BRANCH_NAME}' mapped to namespace '${env.NAMESPACE}'"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${ECR_REPO}:${IMAGE_TAG} ."
            }
        }

        stage('Push to ECR') {
            steps {
                sh """
                    aws ecr get-login-password --region ${AWS_REGION} \
                    | docker login --username AWS --password-stdin ${ECR_REPO}
                    docker push ${ECR_REPO}:${IMAGE_TAG}
                """
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: "${KUBE_CONFIG_CREDENTIAL_ID}", variable: 'KUBECONFIG')]) {
                    sh """
                        export KUBECONFIG=$KUBECONFIG
                        echo "Deploying to Kubernetes namespace: ${NAMESPACE}"

                        # Replace image in deployment file
                        sed -i "s|IMAGE_PLACEHOLDER|${ECR_REPO}:${IMAGE_TAG}|g" k8s/Deployment.yaml

                        # Apply manifests
                        kubectl apply -n ${NAMESPACE} -f k8s/
                    """
                }
            }
        }
    }
}
