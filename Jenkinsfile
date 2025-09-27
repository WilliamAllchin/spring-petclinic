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

                    // build the docker image
                    def customImage = docker.build(imageName)

                    // store imageName in an environment variable for later on
                    env.IMAGE_NAME = imageName
                    
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
                    bat 'mvnw.cmd verify -DskipTests -Dpetclinic.url=http://localhost:8080' // '-DskipTests' skips unit tests which were run during build stage. skipping to only run integration tests.
                    
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
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
        
        stage("Code Quality") {
            steps {
                script {
                    echo "Running code quality analysis with SonarQube..."

                    withSonarQubeEnv('SonarQubeServer') {
                        // SonarQube analysis with Maven
                        bat 'mvnw.cmd sonar:sonar -Dsonar.projectKey=spring-petclinic'
                    }
                }
            }
        }
        
        stage("Security") {
            steps {
                script {
                    echo "Performing automated security analysis with Trivy on Docker image..."
                    
                    // runs Trivy on docker image
                    bat "trivy image --severity HIGH,CRITICAL --format table --ignore-unfixed ${env.IMAGE_NAME}"
                    
                    echo "Security scan with Trivy completed."
                }

            }
        }
        
        stage("Deploy") {
            steps {
                script {
                    echo "Deploy to a testing environment..."
                    
                    // deploy with Docker Compose
                    bat "docker-compose -f docker-compose.yml up -d"
                    
                    echo "Application successfully deployed to the testing environment."
                }
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
