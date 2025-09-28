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
                    
                    // compiles code and packages into a .jar
                    bat 'mvnw.cmd clean package -DskipTests' // '-DskipTests' skips unit tests which were will be run during Test stage
                    
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
                    echo "Starting database services for integration tests..."
                    bat 'docker-compose -f docker-compose.yml up -d'
                    
                    // wait for services to be ready
                    sleep(time: 30, unit: 'SECONDS')
                    
                    echo "Running JUnit and integration tests..."
                    bat 'mvnw.cmd test -Dspring.profiles.active=postgres'
                }
            }
            post {
                always {
                    // stop database services
                    bat 'docker-compose -f docker-compose.yml down'
                    junit 'target/surefire-reports/TEST-*.xml' // save results
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
                script {
                    echo "Promoting application to a production environment..."
                    
                    withAWS(region: "${env.AWS_REGION}", credentials: 'aws-creds'){

                        // get ECR login and login to Docker
                        bat """
                            aws ecr get-login-password --region ${env.AWS_REGION} | docker login --username AWS --password-stdin 262740797964.dkr.ecr.ap-southeast-2.amazonaws.com
                        """

                        // tag image with ECR repo url
                        def ecrImageName = "262740797964.dkr.ecr.ap-southeast-2.amazonaws.com/${env.REPO_NAME}:${env.BUILD_ID}"
                        bat "docker tag ${env.IMAGE_NAME} ${ecrImageName}"

                        // tag as latest
                        def ecrImageLatest = "262740797964.dkr.ecr.ap-southeast-2.amazonaws.com/${env.REPO_NAME}:latest"
                        bat "docker tag ${env.IMAGE_NAME} ${ecrImageLatest}"

                        // push tags to ECR
                        bat "docker push ${ecrImageName}"
                        bat "docker push ${ecrImageLatest}"

                        echo "Successfully pushed image to ECR."
                    }
                }
            }
        }
        
        stage("Monitoring") {
            steps {
                script {
                    echo "Monitoring application in production..."

                    withAWS(region: "${env.AWS_REGION}", credentials: 'aws-creds') {
                
                        echo "Checking ECR exists..."
                        bat """
                            aws ecr describe-images ^
                                --repository-name ${env.REPO_NAME} ^
                                --image-ids imageTag=latest ^
                                --region ${env.AWS_REGION}
                        """
                        
                        echo "Setting up CloudWatch monitoring..."
                        bat """
                            aws cloudwatch put-metric-data ^
                                --namespace "SpringPetClinic" ^
                                --metric-data MetricName=ProductionDeployment,Value=1,Unit=Count
                        """
                    }
                }
            }
        }
        
    }
}
