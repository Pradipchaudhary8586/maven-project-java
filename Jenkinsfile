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
                copyArtifacts filter: '**/*.war', fingerprintArtifacts: true, projectName: env.JOB_NAME, selector: specific(env.BUILD_NUMBER)
                echo "Creating Docker image..."
                sh 'docker build -t $dockerImage:$BUILD_NUMBER .'
            }
        }

        stage('Trivy Scan for Docker Image') {
            steps {
                echo "Scanning Docker image..."
                sh 'trivy image $dockerImage:$BUILD_NUMBER || true'
            }
        }

        stage('Push Image to DockerHub') {
            agent { label 'JenkinsSlave' }
            steps {
                withDockerRegistry([credentialsId: 'dockerhub-credentials', url: '']) {
                    sh 'docker push $dockerImage:$BUILD_NUMBER'
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
}
