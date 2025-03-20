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
                    echo "Build completed, archiving the war file..."
                    archiveArtifacts artifacts: '**/*.war', followSymlinks: false
                }
            }
        }

        stage('Create Docker Image') {
            steps {
                copyArtifacts projectName: env.JOB_NAME, filter: '**/*.war', fingerprint: true, selector: lastSuccessful()
                echo "Creating Docker image..."
                sh '''
                docker build -t $dockerImage:$BUILD_NUMBER .
                docker tag $dockerImage:$BUILD_NUMBER $dockerImage:latest
                '''
            }
        }

        stage('Trivy Scan for Docker Image') {
            steps {
                echo "Scanning Docker image..."
                sh 'trivy image --exit-code 0 $dockerImage:$BUILD_NUMBER || true'
            }
        }

        stage('Push Image to DockerHub') {
            steps {
                withDockerRegistry([credentialsId: 'dockerhub-credentials', url: 'https://index.docker.io/v1/']) {
                    sh '''
                    docker push $dockerImage:$BUILD_NUMBER
                    docker push $dockerImage:latest
                    '''
                }
            }
        }

        stage('Deploy to Development Env') {
            steps {
                echo "Deploying app to development environment..."
                sh '''
                docker stop tomcatInstanceDev || true
                docker rm tomcatInstanceDev || true
                docker run -itd --name tomcatInstanceDev -p 8082:8080 $dockerImage:$BUILD_NUMBER
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
                docker run -itd --name tomcatInstanceProd -p 8083:8080 $dockerImage:$BUILD_NUMBER
                '''
            }
        }
    }

    post { 
        always { 
            mail to: 'iampradip.creation@gmail.com',
            subject: "Job '${JOB_NAME}' (${BUILD_NUMBER}) status",
            body: "Please go to ${BUILD_URL} and verify the build"
        }

        success {
            mail bcc: '', body: """Hi Team,
            Build #$BUILD_NUMBER is successful, please go through the URL:
            $BUILD_URL
            and verify the details.
            Regards,
            DevOps Team""", cc: '', from: '', replyTo: '', subject: 'BUILD SUCCESS NOTIFICATION', to: 'iampradip.creation@gmail.com'
        }

        failure {
            mail bcc: '', body: """Hi Team,
            Build #$BUILD_NUMBER is unsuccessful, please go through the URL:
            $BUILD_URL
            and verify the details.
            Regards,
            DevOps Team""", cc: '', from: '', replyTo: '', subject: 'BUILD FAILED NOTIFICATION', to: 'iampradip.creation@gmail.com'
        }
    }
}
