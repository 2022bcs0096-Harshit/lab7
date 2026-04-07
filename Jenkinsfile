pipeline {
    agent any

    environment {
        IMAGE_NAME = 'harshithbcs96/harshith-2022bcs0096-lab6:latest'
        CONTAINER_NAME = 'lab7-validation-container'
        API_PORT = '8000'
    }

    stages {
        stage('Pull Image') {
            steps {
                echo "Downloading the ML inference Docker image from Docker Hub..."
                sh 'docker pull $IMAGE_NAME'
            }
        }

        stage('Run Container') {
            steps {
                echo "Starting the inference container locally for testing..."
                sh 'docker rm -f $CONTAINER_NAME || true'
                sh 'docker run -d -p $API_PORT:$API_PORT --name $CONTAINER_NAME $IMAGE_NAME'
            }
        }

        stage('Wait for Service Readiness') {
            steps {
                echo "Waiting for API to boot up..."
                timeout(time: 30, unit: 'SECONDS') {
                    waitUntil {
                        script {
                            // Extract IP using simple grep/cut to avoid Groovy string interpolation breaking Docker format braces
                            def API_IP = sh(script: "docker inspect $CONTAINER_NAME | grep -m1 'IPAddress' | cut -d '\"' -f 4", returnStdout: true).trim()
                            
                            def status = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://${API_IP}:${API_PORT}/", returnStdout: true).trim()
                            return status == '200'
                        }
                    }
                }
                echo "API is online and ready for validation!"
            }
        }

        stage('Send Valid Inference Request') {
            steps {
                echo "Sending valid test data to /predict endpoint..."
                script {
                    def API_IP = sh(script: "docker inspect $CONTAINER_NAME | grep -m1 'IPAddress' | cut -d '\"' -f 4", returnStdout: true).trim()
                    
                    def response = sh(script: "curl -s -w '\\nHTTP_STATUS:%{http_code}' -X POST http://${API_IP}:${API_PORT}/predict -H 'Content-Type: application/json' -d @valid_input.json", returnStdout: true).trim()
                    echo "Response: ${response}"
                    
                    if (!response.contains('HTTP_STATUS:200')) {
                        error "FAIL: Valid test request did not return a successful 200 HTTP code!"
                    }
                    if (!response.contains('"prediction"')) {
                        error "FAIL: The response format is incorrect. 'prediction' key is missing!"
                    }
                    echo "PASS: Pipeline correctly validated standard ML inputs!"
                }
            }
        }

        stage('Send Invalid Request') {
            steps {
                echo "Testing Error Handling mechanism with malformed input data..."
                script {
                    def API_IP = sh(script: "docker inspect $CONTAINER_NAME | grep -m1 'IPAddress' | cut -d '\"' -f 4", returnStdout: true).trim()
                    
                    def response = sh(script: "curl -s -w '\\nHTTP_STATUS:%{http_code}' -X POST http://${API_IP}:${API_PORT}/predict -H 'Content-Type: application/json' -d @invalid_input.json", returnStdout: true).trim()
                    echo "Bad Input Response: ${response}"
                    
                    if (response.contains('HTTP_STATUS:200')) {
                        error "FAIL: API blindly accepted terrible input data without throwing an error!"
                    }
                    echo "PASS: API successfully rejected the bad data with a validation error!"
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline complete. Tearing down local test container..."
            sh 'docker stop $CONTAINER_NAME || true'
            sh 'docker rm -f $CONTAINER_NAME || true'
        }
        success {
            echo "CI/CD Model Validation Status: ALL TESTS PASSED SUCCESSFULLY!"
        }
        failure {
            echo "CI/CD Model Validation Status: PIPELINE FAILED DUE TO FAILED TEST OR BAD IMAGE!"
        }
    }
}
