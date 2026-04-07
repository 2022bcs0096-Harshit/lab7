pipeline {
    agent any

    environment {
        IMAGE_NAME = 'harshithbcs96/harshith-2022bcs0096-lab7:latest'
        CONTAINER_NAME = 'lab7-validation-container'
        API_PORT = '8000'
    }

    stages {
        stage('Pull Image') {
            steps {
                echo "=============================================="
                echo " Using locally built ML inference Docker image"
                echo " Image: ${IMAGE_NAME}"
                echo "=============================================="
                sh 'docker images | grep harshith-2022bcs0096-lab7 || echo "WARNING: Image not found locally!"'
            }
        }

        stage('Run Container') {
            steps {
                echo "Starting the inference container locally for testing..."
                sh 'docker rm -f $CONTAINER_NAME || true'
                sh 'docker run -d --name $CONTAINER_NAME $IMAGE_NAME'
                sh 'sleep 5'
            }
        }

        stage('Wait for Service Readiness') {
            steps {
                echo "Waiting for API to boot up..."
                script {
                    def API_IP = sh(script: 'docker exec $CONTAINER_NAME hostname -i', returnStdout: true).trim()
                    echo "Container IP: ${API_IP}"

                    timeout(time: 30, unit: 'SECONDS') {
                        waitUntil {
                            script {
                                def code = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://${API_IP}:${API_PORT}/", returnStdout: true).trim()
                                echo "Health check status: ${code}"
                                return code == '200'
                            }
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
                    def API_IP = sh(script: 'docker exec $CONTAINER_NAME hostname -i', returnStdout: true).trim()

                    def response = sh(script: """curl -s -w '\\nHTTP_STATUS:%{http_code}' -X POST http://${API_IP}:${API_PORT}/predict -H 'Content-Type: application/json' -d @valid_input.json""", returnStdout: true).trim()
                    echo "Response: ${response}"

                    if (!response.contains('HTTP_STATUS:200')) {
                        error "FAIL: Valid test request did not return HTTP 200!"
                    }
                    if (!response.contains('"prediction"')) {
                        error "FAIL: Response missing 'prediction' key!"
                    }
                    echo "PASS: Valid inference request returned correct prediction!"
                }
            }
        }

        stage('Send Invalid Request') {
            steps {
                echo "Testing error handling with malformed input..."
                script {
                    def API_IP = sh(script: 'docker exec $CONTAINER_NAME hostname -i', returnStdout: true).trim()

                    def response = sh(script: """curl -s -w '\\nHTTP_STATUS:%{http_code}' -X POST http://${API_IP}:${API_PORT}/predict -H 'Content-Type: application/json' -d @invalid_input.json""", returnStdout: true).trim()
                    echo "Bad Input Response: ${response}"

                    if (response.contains('HTTP_STATUS:200')) {
                        error "FAIL: API accepted invalid data without error!"
                    }
                    echo "PASS: API correctly rejected malformed input!"
                }
            }
        }
    }

    post {
        always {
            echo "Tearing down test container..."
            sh 'docker stop $CONTAINER_NAME || true'
            sh 'docker rm -f $CONTAINER_NAME || true'
        }
        success {
            echo "=========================================="
            echo " ALL VALIDATION TESTS PASSED SUCCESSFULLY"
            echo "=========================================="
        }
        failure {
            echo "=========================================="
            echo " PIPELINE FAILED - VALIDATION ERROR"
            echo "=========================================="
        }
    }
}
