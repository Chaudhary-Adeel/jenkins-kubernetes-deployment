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
                    // Python script to perform DAST scanning
                    def pythonScript = """
import requests
import json
import time

scan_payload = {{
    'urls': ['http://localhost:3000/']
}}

headers = {{
    'Content-Type': 'application/json',
    'Accept': 'application/json'
}}

# Perform HTTP POST request to initiate the scan
response = requests.post('${env.BURP_BASE_URL}/scan', headers=headers, json=scan_payload)

if response.status_code != 201:
    raise Exception('Failed to trigger Burp Suite scan. HTTP status %s' % response.status_code)

# Extract task ID from response headers
task_id = response.headers.get('Location', '').split('/')[-1]

print('Scan successfully started. Task ID: %s' % task_id)

# Polling to check scan status
scan_status = 'initializing'
max_attempts = 60
attempts = 0

while scan_status not in ['succeeded', 'failed'] and attempts < max_attempts:
    time.sleep(30)
    progress_response = requests.get('${env.BURP_BASE_URL}/scan/%s' % task_id, headers=headers)
    
    if progress_response.status_code == 200:
        scan_status = progress_response.json().get('scan_status')
        print('Scan Status: %s' % scan_status)
    else:
        raise Exception('Failed to fetch scan progress. HTTP status %s' % progress_response.status_code)
    
    attempts += 1

if scan_status != 'succeeded':
    raise Exception('Burp Suite scan did not complete successfully. Final status: %s' % scan_status)

print('Burp Suite scan completed successfully!')
"""

                    // Write the Python script to a temporary file
                    def scriptFile = writeFile(file: 'dast_scan.py', text: pythonScript)

                    // Execute the Python script using python
                    bat "python ${scriptFile}"
                }
            }
        }
    }
}
