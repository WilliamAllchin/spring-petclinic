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

                    //  delete leftover files from previous builds to prevent checkstyle errors from Maven
                    bat "del ssh-key.pem 2>nul || echo No ssh-key.pem to delete"
                    bat "del petclinic-image.tar 2>nul || echo No tar file to delete"

                    // clean and compile code
                    bat 'mvnw.cmd clean compile'
                    
                    // package application
                    bat 'mvnw.cmd package -DskipTests' // '-DskipTests' skips unit tests which will be run during Test stage
                    
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
                    echo "Running tests..."

                    // run unit tests
                    bat '''
                        mvnw.cmd test -Dtest="!*IntegrationTests" ^
                        -DfailIfNoTests=false ^
                        -Dspring.profiles.active=test ^
                        -Dspring.datasource.url=jdbc:h2:mem:testdb ^
                        -Dspring.datasource.driver-class-name=org.h2.Driver ^
                        -Dspring.jpa.database-platform=org.hibernate.dialect.H2Dialect ^
                        -Dspring.docker.compose.enabled=false
                    '''
                    
                    echo "Tests completed successfully!"
                }
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'target/surefire-reports/TEST-*.xml' // save results
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

                    // stop existing containers
                    bat "docker-compose down || true"
                    
                    // start all services by passing BUILD_ID to docker-compose
                    withEnv(["BUILD_ID=${env.BUILD_ID}"]) {
                        bat "docker-compose up -d"
                    }
                    
                    echo "Application deploying to the testing environment locally at port 8090."
                }
            }
        }
        
        stage("Release") {
            steps {
                script {
                    echo "Promoting application to a production environment..."

                    // save image as .tar for promotion to EC2 instance
                    bat "docker save ${env.IMAGE_NAME} -o petclinic-image.tar"

                    sshPublisher(
                        publishers: [
                            sshPublisherDesc(
                                configName: 'ec2-production',
                                transfers: [
                                    sshTransfer(
                                        sourceFiles: 'petclinic-image.tar',
                                        execCommand: """
                                            echo "Deploying ${env.IMAGE_NAME}..."
                                            
                                            # load new image
                                            sudo docker load -i petclinic-image.tar
                                            
                                            # replace running container
                                            sudo docker stop petclinic || echo "No container to stop"
                                            sudo docker rm petclinic || echo "No container to remove"
                                            sudo docker run -d --name petclinic -p 8080:8080 ${env.IMAGE_NAME}
                                            
                                            # cleanup and verify
                                            rm -f petclinic-image.tar
                                            sudo docker ps | grep petclinic
                                            
                                            echo "Deployment complete!"
                                        """
                                    )
                                ]
                            )
                        ]
                    )

                    // deletes .tar after promoting it to production
                    bat "del petclinic-image.tar"

                    echo "Production deployment accessible at EC2 instance 3.25.139.255 on port 8080"
                }
            }
        }
        
        stage("Monitoring") {
            steps {
                script {
                    echo "Installing Datadog Agent on EC2..."
        
                    withCredentials([string(credentialsId: 'datadog-api-key', variable: 'DD_API_KEY')]) {
                        
                        // install Datadog agent on EC2
                        sshPublisher(
                            publishers: [
                                sshPublisherDesc(
                                    configName: 'ec2-production',
                                    transfers: [
                                        sshTransfer(
                                            sourceFiles: '',
                                            execCommand: """
                                                echo "Cleaning up existing Datadog containers."
                                                sudo docker stop datadog-agent || echo "No datadog-agent container to stop"
                                                sudo docker rm datadog-agent || echo "No datadog-agent container to remove"
                                            
                                                # install Datadog agent
                                                DD_API_KEY=${DD_API_KEY} bash -c "\$(curl -L https://s3.amazonaws.com/dd-agent/scripts/install_script.sh)"
                                                
                                                # configure Docker monitoring
                                                sudo docker run -d --name datadog-agent \\
                                                    -v /var/run/docker.sock:/var/run/docker.sock:ro \\
                                                    -v /proc/:/host/proc/:ro \\
                                                    -v /sys/fs/cgroup/:/host/sys/fs/cgroup:ro \\
                                                    -e DD_API_KEY=${DD_API_KEY} \\
                                                    -e DD_SITE="ap2.datadoghq.com" \\
                                                    datadog/agent:latest

                                                echo "Verifying Datadog installation."
                                                sleep 5
                                                sudo docker ps | grep datadog-agent || echo "Datadog container not found."
                                                echo "Datadog monitoring configured successfully."
                                            """
                                        )
                                    ]
                                )
                            ]
                        )
        
                        echo "Datadog agent installed successfully."
                    }
                }
            }
        }
        
    }
}
