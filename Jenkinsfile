pipeline {
    environment {
        dockerimagename = "chaudharyadeel/react-app"
        dockerImage = ""
        KUBECONFIG = credentials('kubeconfig-credential-id') // The credential ID for your Kubernetes config
    }

    agent any

    stages {
        stage('Checkout Source') {
            steps {
                git branch: 'main', url: 'https://github.com/chaudhary-adeel/jenkins-kubernetes-deployment.git'
            }
        }

        stage('Build image') {
            steps {
                script {
                    dockerImage = docker.build dockerimagename
                }
            }
        }

        stage('Pushing Image') {
            environment {
                registryCredential = 'docker-hub'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', registryCredential) {
                        dockerImage.push("latest")
                    }
                }
            }
        }

        stage('Deploying React.js container to Kubernetes') {
            steps {
                script {
                    // Apply the deployment configuration
                    sh 'kubectl apply -f deployment.yaml --kubeconfig=$KUBECONFIG'
                    // Apply the service configuration
                    sh 'kubectl apply -f service.yaml --kubeconfig=$KUBECONFIG'
                }
            }
        }
    }
}
