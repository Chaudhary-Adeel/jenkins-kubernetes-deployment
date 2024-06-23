pipeline {
    environment {
        dockerimagename = "chaudharyadeel/react-app"
        dockerImage = ""
        KUBECONFIG = credentials('kubeconfig-credential-id') // The credential ID for your Kubernetes config
        SNYK_TOKEN = credentials('snyk-token-id') // The credential ID for your Snyk API token
    }

    agent any

    stages {
        stage('Checkout Source') {
            steps {
                git branch: 'main', url: 'https://github.com/chaudhary-adeel/jenkins-kubernetes-deployment.git'
            }
        }

        stage('SAST and Secrets Scanning with Snyk') {
            steps {
                script {
                    // Authenticate Snyk
                    sh 'snyk auth ${SNYK_TOKEN}'
                    
                    // Perform SAST and Secrets Scanning using Snyk
                    sh 'snyk code test --severity-threshold=high || true'
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
                    sh "snyk container test ${dockerimagename} --severity-threshold=high || true"
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
                    // Apply the deployment configuration
                    bat 'kubectl apply -f deployment.yaml --kubeconfig=%KUBECONFIG%'
                    // Apply the service configuration
                    bat 'kubectl apply -f service.yaml --kubeconfig=%KUBECONFIG%'
                }
            }
        }

        stage('DAST Scanning') {
            steps {
                script {
                    // Trigger Burp Suite scan
                    def burpSuiteScanResponse = httpRequest httpMode: 'POST', 
                        acceptType: 'APPLICATION_JSON',
                        contentType: 'APPLICATION_JSON',
                        url: 'http://127.0.0.1:1337/v0.1/scans',
                        requestBody: """{
                            "urls": ["http://localhost:3000/"],
                            "name": "React App DAST Scan",
                            "scope": null,
                            "application_logins": [],
                            "scan_configurations": [],
                            "resource_pool": null,
                            "scan_callback": null,
                            "protocol_option": null
                        }"""

                    def scanId = readJSON text: burpSuiteScanResponse.content
                    echo "Burp Suite Scan initiated with ID: ${scanId}"

                    // Poll for scan completion
                    def scanCompleted = false
                    while (!scanCompleted) {
                        sleep(time: 60, unit: 'SECONDS') // Wait for 1 minute before polling again

                        def scanStatusResponse = httpRequest httpMode: 'GET', 
                            acceptType: 'APPLICATION_JSON',
                            url: "http://127.0.0.1:1337/v0.1/scans/${scanId}"

                        def scanStatus = readJSON text: scanStatusResponse.content
                        echo "Scan status: ${scanStatus.state}"
                        scanCompleted = (scanStatus.state == 'completed')
                    }

                    // Fetch scan results
                    def scanResultsResponse = httpRequest httpMode: 'GET',
                        acceptType: 'APPLICATION_JSON',
                        url: "http://127.0.0.1:1337/v0.1/scans/${scanId}/report"

                    def scanResults = readJSON text: scanResultsResponse.content
                    echo "Scan results: ${scanResults}"

                    // Optionally fail the build if vulnerabilities are found
                    if (scanResults.vulnerabilities.size() > 0) {
                        error("DAST scan found vulnerabilities")
                    }
                }
            }
        }
    }
}
