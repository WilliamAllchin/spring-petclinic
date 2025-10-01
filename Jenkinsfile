pipeline {
    agent any

    tools {
        maven 'Maven3' 
        jdk 'JDK17' // using Java 17
    }

    environment {
        AWS_REGION = 'ap-southeast-2'
        REPO_NAME = 'sit753/task73hd'
    }
    
    stages {
        stage("Build") {
            steps {
                script {
                    echo "Building using Maven..."

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
                    
                    // start all services
                    bat "docker-compose up -d"
                    
                    echo "Application successfully deployed to the testing environment locally at port 8090."
                }
            }
        }
        
        stage("Release") {
            steps {
                script {
                    echo "Promoting application to a production environment..."

                    // save image as .tar for promotion to EC2 instance
                    bat "docker save ${env.IMAGE_NAME} -o petclinic-image.tar"

                    sshagent(['ec2-deploy-key']) {
                        // Transfer and deploy to EC2
                        bat "scp -o StrictHostKeyChecking=no petclinic-image.tar ec2-user@3.25.139.255:/home/ec2-user/"
                    
                        bat """
                            ssh -o StrictHostKeyChecking=no ec2-user@3.25.139.255 "sudo docker load -i petclinic-image.tar && sudo docker stop petclinic || true && sudo docker rm petclinic || true && sudo docker run -d --name petclinic -p 8080:8080 ${env.IMAGE_NAME}"
                        """
                    }

                    // deletes .tar after promoting it to production
                    bat "del petclinic-image.tar"

                    echo "Production deployment accessible at EC2 instance 3.25.139.255 on port 8080"
                }
            }
        }
        
        stage("Monitoring") {
            steps {
                script {
                    echo "Setting up production application monitoring..."
        
                    withCredentials([string(credentialsId: 'datadog-api-key', variable: 'DD_API_KEY')]) {
                        withAWS(region: "${env.AWS_REGION}", credentials: 'aws-creds') {
        
                            // get image size in bytes from AWS
                            def imageSizeOutput = bat(
                                script: """
                                    @echo off
                                    aws ecr describe-images ^
                                        --repository-name ${env.REPO_NAME} ^
                                        --image-ids imageTag=${env.BUILD_ID} ^
                                        --query "imageDetails[0].imageSizeInBytes" ^
                                        --output text
                                """,
                                returnStdout: true
                            ).trim()
                            
                            // extract only the numeric value from last line of output
                            def imageSizeBytes = imageSizeOutput.split('\n')[-1].trim()
                            
                            def imageSizeMB = (imageSizeBytes as Long) / 1048576 // converts bytes to MB
                            def currentTime = System.currentTimeMillis() / 1000
                            
                            echo "Sending metrics to Datadog..."
                            echo "Image size: ${imageSizeMB} MB"

                            // send metrics to Datadog
                            bat """
                                powershell -Command "\$headers = @{'DD-API-KEY'='${DD_API_KEY}'; 'Content-Type'='application/json'}; \$body = @{series=@(@{metric='ecr.image.size'; points=@(@(${currentTime}, ${imageSizeMB})); type='gauge'; tags=@('repository:${env.REPO_NAME}','build:${env.BUILD_NUMBER}')})} | ConvertTo-Json -Depth 4; Invoke-RestMethod -Uri 'https://api.ap2.datadoghq.com/api/v1/series' -Method Post -Headers \$headers -Body \$body"
                            """
        
                            // send deployment event
                            bat """
                                powershell -Command "\$headers = @{'DD-API-KEY'='${DD_API_KEY}'; 'Content-Type'='application/json'}; \$body = @{title='ECR Deployment: ${env.REPO_NAME}'; text='Deployed build ${env.BUILD_NUMBER} to ECR'; tags=@('repository:${env.REPO_NAME}','build:${env.BUILD_NUMBER}'); alert_type='success'} | ConvertTo-Json; Invoke-RestMethod -Uri 'https://api.ap2.datadoghq.com/api/v1/events' -Method Post -Headers \$headers -Body \$body"
                            """
        
                            echo "Datadog metrics sent successfully!"
                        }
                    }
                }
            }
        }
        
    }
}
