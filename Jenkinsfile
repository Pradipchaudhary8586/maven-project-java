pipeline {
    agent any
    environment {
        dockerImage = "pradipchaudhary7/jenkinspipelinesetup"
    }
    stages {
        stage('Build Java App') {
            steps {
                sh 'mvn -f pom.xml clean package'
            }
            post {
                success {
                    echo "Build completed, archiving the WAR file..."
                    archiveArtifacts artifacts: '**/*.war', followSymlinks: false
                }
            }
        }

        stage('Create Docker Image') {
            steps {
                echo "Creating Docker image..."
                sh "docker build -t ${dockerImage}:${BUILD_NUMBER} ."
            }
        }

        stage('Trivy Scan for Docker Image') {
            steps {
                echo "Scanning Docker image for vulnerabilities..."
                sh '''
                trivy image --timeout 10m --scanners vuln --exit-code 1 --severity HIGH,CRITICAL --ignore-unfixed ${dockerImage}:${BUILD_NUMBER} || echo "No Critical vulnerabilities found"
                '''
            }
        }

        stage('Push Image to DockerHub') {
            steps {
                echo "Logging in to DockerHub..."
                withDockerRegistry([credentialsId: 'dockerhub-credentials', url: 'https://index.docker.io/v1/']) {
                    sh "docker push ${dockerImage}:${BUILD_NUMBER}"
                }
            }
        }

        stage('Deploy to Development Env') {
            steps {
                echo "Deploying app to development environment..."
                sh '''
                docker stop tomcatInstanceDev || true
                docker rm tomcatInstanceDev || true
                docker run -itd --name tomcatInstanceDev -p 8082:8080 ${dockerImage}:${BUILD_NUMBER}
                '''
            }
        }

        stage('Deploy to Production Env') {
            steps {
                timeout(time: 1, unit: 'DAYS') {
                    input message: 'Approve PRODUCTION Deployment?'
                }
                echo "Deploying app to production environment..."
                sh '''
                docker stop tomcatInstanceProd || true
                docker rm tomcatInstanceProd || true
                docker run -itd --name tomcatInstanceProd -p 8083:8080 ${dockerImage}:${BUILD_NUMBER}
                '''
            }
        }
    }
    post { 
        always { 
            mail to: 'iampradip.creation@gmail.com',
            subject: "Job '${JOB_NAME}' (${BUILD_NUMBER}) status",
            body: "Please go to ${BUILD_URL} and verify the build."
        }

        success {
            mail body: """Hi Team,
            Build #${BUILD_NUMBER} is successful. Please verify at:
            ${BUILD_URL}
            Regards,
            DevOps Team""", subject: 'BUILD SUCCESS NOTIFICATION', to: 'iampradip.creation@gmail.com'
        }

        failure {
            mail body: """Hi Team,
            Build #${BUILD_NUMBER} failed. Please check logs:
            ${BUILD_URL}
            Regards,
            DevOps Team""", subject: 'BUILD FAILED NOTIFICATION', to: 'iampradip.creation@gmail.com'
        }
    }
}
