pipeline {
    agent any

    environment {
        REGION = 'ap-northeast-2'
        EKS_API = 'EKS Cluster Api Server 엔드포인트'
        EKS_CLUSTER_NAME = 'EKS Cluster 이름'
        EKS_JENKINS_CREDENTIAL_ID = 'Kubernetes Credential ID'
        ECR_PATH = 'ECR Repository URI 작성 /repository-name은 제거'
        ECR_IMAGE = '이미지 이름'
        AWS_CREDENTIAL_ID = 'AWS Credential ID'
    }
    stages {
        stage('Clone Repository') {
            steps {
                checkout scm
            }
        }
        stage('Maven Build') {
            steps {
                sh 'mvn package -DskipTests'
            }
        }
        stage('Docker Build') {
            steps {
                script {
                    docker.withRegistry("https://${ECR_PATH}", "ecr:${REGION}:${AWS_CREDENTIAL_ID}") {
                        image = docker.build("${ECR_PATH}/${ECR_IMAGE}")
                    }
                }
            }
        }
        stage('Push to ECR') {
            steps {
                script {
                    docker.withRegistry("https://${ECR_PATH}", "ecr:${REGION}:${AWS_CREDENTIAL_ID}") {
                        image.push("v${env.BUILD_NUMBER}")
                    }
                }
            }
        }
        stage('CleanUp Images') {
            steps {
                sh """
                docker rmi ${ECR_PATH}/${ECR_IMAGE}:v$BUILD_NUMBER
                docker rmi ${ECR_PATH}/${ECR_IMAGE}:latest
                """
            }
        }
        stage('Deploy to k8s') {
            steps {
                withKubeConfig([credentialsId: EKS_JENKINS_CREDENTIAL_ID,
                                serverUrl: EKS_API,
                                clusterName: EKS_CLUSTER_NAME]) {
                    sh "sed 's/latest/v${env.BUILD_ID}/g' kubernetes/deploy.yaml > output.yaml"
                    sh "aws eks --region ${REGION} update-kubeconfig --name ${EKS_CLUSTER_NAME}"
                    sh "kubectl apply -f output.yaml"
                    sh "kubectl apply -f kubernetes/service.yaml"
                    sh "rm output.yaml"
                }
            }
        }
    }
}
