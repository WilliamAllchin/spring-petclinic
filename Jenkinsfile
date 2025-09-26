pipeline {
    agent any
    
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
                echo "Running automated tests..."
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
