pipeline {
    environment {
        dockerimagename = "chaudharyadeel/react-app"
        dockerImage = ""
        KUBECONFIG = credentials('kubeconfig-credential-id') // The credential ID for your Kubernetes config
        SNYK_TOKEN = credentials('snyk-token-id') // The credential ID for your Snyk API token
        BURP_BASE_URL = 'http://127.0.0.1:1337/v0.1'
    }

    agent any

    stages {
        stage('SAST and Secrets Scanning with Snyk') {
            steps {
                script {
                    // Authenticate Snyk
                    bat "snyk auth %SNYK_TOKEN%"
                    
                    // Perform SAST and Secrets Scanning using Snyk
                    bat "snyk code test --severity-threshold=high || exit 0"
                }
            }
        }

        stage('Build Image') {
            steps {
                script {
                    dockerImage = docker.build(dockerimagename, '--pull .')
                }
            }
        }

        stage('Container Scanning with Snyk') {
            steps {
                script {
                    // Scan the Docker image using Snyk
                    bat "snyk container test ${dockerimagename} --severity-threshold=high || exit 0"
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

        stage('Deploying React.js Container to Kubernetes') {
            steps {
                script {
                    // Set the KUBECONFIG environment variable within the bat command
                    bat '''
                        set KUBECONFIG=%KUBECONFIG%
                        kubectl apply -f deployment.yaml
                        kubectl apply -f service.yaml
                    '''
                }
            }
        }

        stage('DAST Scanning') {
            steps {
                script {
                    def scanPayload = [
                        urls: ["http://localhost:3000/"],  // Replace with the URLs you want to scan
                    ]
                    def jsonPayload = new groovy.json.JsonOutput().toJson(scanPayload)
                    echo "Request Payload:: --> ${jsonPayload}"
                    
                    // Run Python script for DAST scan
                    bat """python "${WORKSPACE}\\dast_scan.py" '${jsonPayload}'"""
                }
            }
        }
    }
}
