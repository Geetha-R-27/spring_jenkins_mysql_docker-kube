pipeline {
    agent any
    tools{
        maven 'maven'
    }
    environment {
        IMAGE_NAME = "geethar27/kubectl"
        IMAGE_TAG  = "latest"
        NAMESPACE  = "dev"
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
    }

    stages {

        stage('Build JAR') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                      echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                      docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    '''
                }
            }
        }

        stage('Deploy MySQL (StatefulSet FIRST)') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-dev', variable: 'KUBECONFIG')]) {
                    sh '''
                      export KUBECONFIG=$KUBECONFIG
                      kubectl get namespace ${NAMESPACE} || kubectl create namespace ${NAMESPACE}
                      kubectl apply -f k8s/mysql-secret.yaml -n ${NAMESPACE}
                      kubectl apply -f k8s/mysql-service.yaml -n ${NAMESPACE}
                      kubectl apply -f k8s/mysql-statefulset.yaml -n ${NAMESPACE}
                    '''
                }
            }
        }

        stage('Deploy Spring Boot (Deployment NEXT)') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-dev', variable: 'KUBECONFIG')]) {
                    sh '''
                      export KUBECONFIG=$KUBECONFIG
                      kubectl apply -f k8s/app-deployment.yaml -n ${NAMESPACE}
                      kubectl apply -f k8s/app-service.yaml -n ${NAMESPACE}
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ CI/CD Pipeline completed successfully"
        }
        failure {
            echo "❌ CI/CD Pipeline failed"
        }
    }
}

