pipeline {
    agent any

    tools {
        maven 'Maven3' 
        jdk 'JDK17' // using Java 17
    }
    
    stages {
        stage("Build") {
            steps {
                script {
                    echo "Building using Maven..."
                    
                    // compiles code and packages into a .jar
                    bat 'mvnw.cmd clean package' // 'clean' removes previous builds
                    
                    // name image with build number
                    def imageName = "petclinic-app:${env.BUILD_ID}"
                    def customImage = docker.build(imageName)
                    
                    echo "Successfully built Docker image ${imageName}."
                }
            }
        }
        
        stage("Test") {
            steps {
                script {
                    // start database and app services from docker-compose.yml
                    echo "Starting integration test environment with Docker Compose..."
                    bat 'docker-compose -f docker-compose.yml up -d --wait'

                    echo "Integration test environment started."

                    // run integration tests with Maven 'verify'
                    echo "Running Maven Integration Tests..."
                    bat 'mvnw.cmd verify -Dpetclinic.url=http://localhost:8080'

                    
                    echo "Integration tests completed successfully!"
                }
            }
            // This 'post' block ensures the environment is always cleaned up.
            post {
                always {
                    script {
                        // stops and removes resources created by docker-compose up
                        bat 'docker-compose -f docker-compose.yml down'
                    }
                }
                success {
                    // saves test results
                    junit 'target/failsafe-reports/*.xml'
                }
            }
        }
        
        stage("Code Quality") {
            steps {
                echo "Running code quality analysis with SonarQube..."
            }
        }
        
        stage("Security") {
            steps {
                echo "Performing automated security analysis..."
            }
        }
        
        stage("Deploy") {
            steps {
                echo "Deploy to a testing environment..."
            }
        }
        
        stage("Release") {
            steps {
                echo "Promoting application to a production environment..."
            }
        }
        
        stage("Monitoring") {
            steps {
                echo "Monitoring application in production..."
            }
        }
        
    }
    
}
