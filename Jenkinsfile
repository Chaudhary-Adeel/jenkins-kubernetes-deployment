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
                    bat 'snyk auth %SNYK_TOKEN%'
                    
                    // Perform SAST and Secrets Scanning using Snyk
                    bat 'snyk code test --severity-threshold=high || exit 0'
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
                    
                    def scanResponse = httpRequest(
                        acceptType: 'APPLICATION_JSON',
                        contentType: 'APPLICATION_JSON',
                        httpMode: 'POST',
                        requestBody: jsonPayload,
                        url: "${env.BURP_BASE_URL}/scan"
                    )
                    
                    if (scanResponse.status != 201) {
                        error "Failed to trigger Burp Suite scan. HTTP status ${scanResponse.status}"
                    }
                    
                    def scanLocation = scanResponse.getResponseHeaders().find { it.key == 'Location' }?.value
                    if (!scanLocation) {
                        error "No 'Location' header found in the response"
                    }
                    
                    def taskId = scanLocation.tokenize('/').last()
                    echo "Scan successfully started. Task ID: ${taskId}"
                    
                    // Polling to check scan status
                    def scanStatus = 'initializing'
                    timeout(time: 30, unit: 'MINUTES') {
                        def maxAttempts = 60  // Example: Poll for up to 30 minutes with 30-second interval
                        def attempts = 0
                        
                        while (scanStatus != 'succeeded' && scanStatus != 'failed' && attempts < maxAttempts) {
                            def scanProgress = httpRequest(
                                acceptType: 'APPLICATION_JSON',
                                contentType: 'APPLICATION_JSON',
                                httpMode: 'GET',
                                url: "${env.BURP_BASE_URL}/scan/${taskId}"
                            )
                            
                            if (scanProgress.status == 200) {
                                scanStatus = scanProgress.data.scan_status
                                echo "Scan Status: ${scanStatus}"
                                if (scanStatus == 'succeeded' || scanStatus == 'failed') {
                                    break
                                }
                            } else {
                                error "Failed to fetch scan progress. HTTP status ${scanProgress.status}"
                            }
                            
                            sleep(time: 30, unit: 'SECONDS')  // Wait before the next poll
                            attempts++
                        }
                    }
                    
                    if (scanStatus != 'succeeded') {
                        error "Burp Suite scan did not complete successfully. Final status: ${scanStatus}"
                    }
                    
                    echo "Burp Suite scan completed successfully!"
                }
            }
        }
    }
}
