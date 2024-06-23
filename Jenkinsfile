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
                    // Prepare scan payload
                    def scanPayload = [
                        urls: ["http://localhost:3000/"]
                    ]
                    def jsonPayload = new groovy.json.JsonOutput().toJson(scanPayload)
                    echo "Request Payload:: --> ${jsonPayload}"
                    
                    // Perform HTTP POST request using curl
                    def curlCommand = """
curl -X POST -H "Content-Type: application/json" -H "Accept: application/json" -d '${jsonPayload}' ${env.BURP_BASE_URL}/scan
"""
                    def response = bat(returnStdout: true, script: curlCommand).trim()
                    
                    // Check if the response contains a task ID
                    def taskId = response.readLines().find { it.startsWith('Location: ') }?.split('/')[-1]?.trim()
                    if (!taskId) {
                        error "Failed to trigger Burp Suite scan. Response: ${response}"
                    }
                    
                    echo "Scan successfully started. Task ID: ${taskId}"
                    
                    // Polling to check scan status
                    def scanStatus = 'initializing'
                    timeout(time: 30, unit: 'MINUTES') {
                        def maxAttempts = 60
                        def attempts = 0
                        
                        while (scanStatus != 'succeeded' && scanStatus != 'failed' && attempts < maxAttempts) {
                            def progressCommand = "curl -s -X GET -H 'Accept: application/json' ${env.BURP_BASE_URL}/scan/${taskId}"
                            def progressResponse = bat(returnStdout: true, script: progressCommand).trim()
                            
                            def progressJson = new groovy.json.JsonSlurper().parseText(progressResponse)
                            scanStatus = progressJson.scan_status
                            
                            echo "Scan Status: ${scanStatus}"
                            
                            if (scanStatus == 'succeeded' || scanStatus == 'failed') {
                                break
                            }
                            
                            sleep(time: 30, unit: 'SECONDS')
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
