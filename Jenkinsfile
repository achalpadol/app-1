def backendImage
def proxyImage

pipeline {

    agent any

    environment {

        DOCKER_REGISTRY = "https://index.docker.io/v1/"

        BACKEND_IMAGE = "achalpadol/app-1-backend"
        PROXY_IMAGE   = "achalpadol/app-1-proxy"

        DOCKER_CREDENTIALS = "02_docker_hub_creds"

        AWS_REGION = "ap-south-1"
        EKS_CLUSTER = "cluster-1"

        DEV_NAMESPACE  = "app-dev"
        QA_NAMESPACE   = "app-qa"
        UAT_NAMESPACE  = "app-uat"
        PROD_NAMESPACE = "app-prod"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Images - DEV') {
            steps {
                script {

                    backendImage = docker.build(
                        "${env.BACKEND_IMAGE}:${env.BUILD_NUMBER}",
                        "./backend"
                    )

                    proxyImage = docker.build(
                        "${env.PROXY_IMAGE}:${env.BUILD_NUMBER}",
                        "./proxy"
                    )
                }
            }
        }

        stage('Push Images - DEV') {
            steps {
                script {

                    docker.withRegistry(
                        env.DOCKER_REGISTRY,
                        env.DOCKER_CREDENTIALS
                    ) {

                        backendImage.push("${env.BUILD_NUMBER}")
                        proxyImage.push("${env.BUILD_NUMBER}")
                    }
                }
            }
        }

        stage('Deploy DEV') {
            steps {
                script {
                    deployToEnvironment(
                        env.DEV_NAMESPACE
                    )
                }
            }
        }

        stage('Deploy QA') {
            steps {
                script {
                    deployToEnvironment(
                        env.QA_NAMESPACE
                    )
                }
            }
        }

        stage('Deploy UAT') {
            steps {
                script {
                    deployToEnvironment(
                        env.UAT_NAMESPACE
                    )
                }
            }
        }

        stage('Production Approval') {
            steps {
                input(
                    message: 'Deploy this image to Production?',
                    ok: 'Deploy to Production'
                )
            }
        }

        stage('Deploy PROD') {
            steps {
                script {
                    deployToEnvironment(
                        env.PROD_NAMESPACE
                    )
                }
            }
        }
    }

    post {

        always {
            cleanWs()
        }

        success {
            echo 'CI/CD pipeline completed successfully.'
        }

        failure {
            echo 'CI/CD pipeline failed.'
        }
    }
}


/*
 * Reusable deployment function
 */
def deployToEnvironment(namespace) {

    echo "Deploying image ${env.BUILD_NUMBER} to ${namespace}"

    sh """
        aws eks update-kubeconfig \
        --region ${env.AWS_REGION} \
        --name ${env.EKS_CLUSTER}

        kubectl create namespace ${namespace} \
        --dry-run=client \
        -o yaml | kubectl apply -f -

        kubectl apply \
        -f kube/ \
        -n ${namespace}

        echo "Updating backend image..."

        kubectl set image deployment/app1-backend \
        backend=${env.BACKEND_IMAGE}:${env.BUILD_NUMBER} \
        -n ${namespace}

        echo "Updating frontend image..."

        kubectl set image deployment/app-1 \
        app-1-cont=${env.PROXY_IMAGE}:${env.BUILD_NUMBER} \
        -n ${namespace}

        echo "Waiting for backend rollout..."

        kubectl rollout status deployment/app1-backend \
        -n ${namespace}

        echo "Waiting for frontend rollout..."

        kubectl rollout status deployment/app-1 \
        -n ${namespace}
    """
}
