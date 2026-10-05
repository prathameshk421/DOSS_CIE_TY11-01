pipeline {
    agent any
    triggers {
        pollSCM('* * * * *')
    }
    environment {
        PATH = "/opt/homebrew/bin:/usr/local/bin:${env.PATH}"
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Test HTML') {
            steps {
                sh 'test -f index.html'
                sh 'grep -qi "<html" index.html'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t devops-demo:latest .'
            }
        }
        stage('Load Image to Minikube') {
            steps {
                sh 'minikube image load devops-demo:latest'
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl apply -f deployment.yaml'
                sh 'kubectl apply -f service.yaml'
                sh 'kubectl rollout restart deployment/devops-demo-deployment'
            }
        }
        stage('Verify') {
            steps {
                sh 'kubectl get pods'
                sh 'kubectl get service'
            }
        }
    }
}
