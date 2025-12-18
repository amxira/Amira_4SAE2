pipeline {
    agent any
    
    tools {
        maven 'Maven'
    }
    
    environment {
        IMAGE_NAME = 'amxira/Amira_4SAE2'
        // SonarQube configuration
        SONARQUBE_SERVER = 'SQ1'
        SONAR_PROJECT_KEY = 'amxira_Amira_4SAE2_460a7286-3d11-4296-ac4b-c67c1c522976'
        SONAR_PROJECT_NAME = 'Amira_4SAE2'
        SONAR_HOST_URL = 'http://192.168.33.10:9000'
    }
    
    triggers {
        githubPush()
    }
    
    stages {
        stage('Checkout') {
            steps {
                deleteDir()
                git branch: 'main', 
                    url: 'https://github.com/amxira/Amira_4SAE2.git'
                sh 'chmod +x mvnw'
            }
        }
        
        stage('Build & Test') {
            steps {
                sh './mvnw clean test'
            }
        }
        
        stage('MVN SONARQUBE') {
            steps {
                withSonarQubeEnv("${env.SONARQUBE_SERVER}") {
                    sh """
                        ./mvnw sonar:sonar \
                            -Dsonar.projectKey=${env.SONAR_PROJECT_KEY} \
                            -Dsonar.projectName=${env.SONAR_PROJECT_NAME} \
                            -Dsonar.host.url=http://192.168.33.10:9000
                    """
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                script {
                    // Give SonarQube time to finalize the task status
                    sleep(time: 5, unit: 'SECONDS')
                }
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Package') {
            steps {
                sh './mvnw package -DskipTests'
            }
        }
        
        stage('Docker Build') {
            steps {
                sh """
                    docker build \
                        -t ${env.IMAGE_NAME}:${env.BUILD_NUMBER} \
                        -t ${env.IMAGE_NAME}:latest \
                        .
                """
            }
        }
        
        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PASS')]) {
                    sh """
                        echo "$DOCKERHUB_PASS" | docker login -u "$DOCKERHUB_USER" --password-stdin
                        docker push ${env.IMAGE_NAME}:${env.BUILD_NUMBER}
                        docker push ${env.IMAGE_NAME}:latest
                        docker logout
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo 'Deploying to Kubernetes...'
                    
                    // Apply MySQL resources
                    sh """
                        kubectl apply -f k8s/mysql-configmap.yaml
                        kubectl apply -f k8s/mysql-secret.yaml
                        kubectl apply -f k8s/mysql-pv.yaml
                        kubectl apply -f k8s/mysql-pvc.yaml
                        kubectl apply -f k8s/mysql-deployment.yaml
                        kubectl apply -f k8s/mysql-service.yaml
                    """
                    
                    // Wait for MySQL to be ready
                    sh 'kubectl wait --for=condition=ready pod -l app=mysql --timeout=120s || true'
                    
                    // Apply Application resources
                    sh """
                        kubectl apply -f k8s/app-configmap.yaml
                        kubectl apply -f k8s/app-secret.yaml
                        kubectl set image deployment/student-management student-management=${env.IMAGE_NAME}:${env.BUILD_NUMBER} --record || kubectl apply -f k8s/app-deployment.yaml
                        kubectl apply -f k8s/app-service.yaml
                    """
                    
                    // Wait for application to be ready
                    sh 'kubectl wait --for=condition=ready pod -l app=student-management --timeout=180s || true'
                    
                    // Show deployment status
                    sh """
                        echo "=== Deployment Status ==="
                        kubectl get pods
                        kubectl get svc
                    """
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    echo 'Verifying deployment...'
                    sh """
                        kubectl get deployment student-management
                        kubectl get pods -l app=student-management
                        kubectl describe service student-management-service
                    """
                }
            }
        }
    }

    

    post {
        success {
            echo 'Build and SonarQube analysis succeeded!'
        }
        failure {
            echo 'Build or SonarQube analysis failed!'
        }
    }
}