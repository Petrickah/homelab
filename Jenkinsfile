pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate manifests') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig-k3s']) {
                    sh 'kubectl apply --dry-run=client -f k8s/vaultwarden/'
                }
            }
        }

        stage('Deploy Vaultwarden') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig-k3s']) {
                    sh 'kubectl apply -f k8s/vaultwarden/'
                }
            }
        }

        stage('Verify rollout') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig-k3s']) {
                    sh 'kubectl rollout status deployment/vaultwarden -n vaultwarden --timeout=60s'
                }
            }
        }
    }

    post {
        success {
            echo 'Deployment finished successfully.'
        }
        failure {
            echo 'Deployment failed - check the logs above.'
        }
    }
}