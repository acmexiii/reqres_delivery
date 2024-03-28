pipeline {
    agent any

    environment {
        REGION = 'ap-northeast-2'
        EKS_API = 'https://A854A56A9E8DC64E6BBFD91B57621678.gr7.ap-northeast-2.eks.amazonaws.com'
        EKS_CLUSTER_NAME = 'test-eks'
        EKS_JENKINS_CREDENTIAL_ID = 'kube-cred'
        ECR_PATH = '879772956301.dkr.ecr.ap-northeast-2.amazonaws.com'
        ECR_IMAGE = 'test-product'
        AWS_CREDENTIAL_ID = 'aws-credential'
    }
    stages {
        stage('Clone Repository') {
            steps {
                checkout scm
            }
        }
        stage('Maven Build') {
            steps {
                withMaven(maven: 'Maven') {
                    sh 'mvn package -DskipTests'
                }
            }
        }
        stage('Docker Build') {
            steps {
                script {
                    image = docker.build("${ECR_PATH}/${ECR_IMAGE}")                  
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
                script {
                    // AWS 자격 증명을 가져오기
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "${AWS_CREDENTIAL_ID}", accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        // Kubernetes에 접근을 위한 kubeconfig 설정
                        withKubeConfig([credentialsId: "${EKS_JENKINS_CREDENTIAL_ID}",
                                        serverUrl: "${EKS_API}",
                                        clusterName: "${EKS_CLUSTER_NAME}"]) {
                            // 배포할 YAML 파일을 빌드 ID에 맞게 수정
                            sh "sed 's/latest/v${env.BUILD_ID}/g' kubernetes/deploy.yaml > output.yaml"
                            // AWS CLI를 사용하여 kubeconfig 설정 업데이트
                            sh "aws eks --region ${REGION} update-kubeconfig --name ${EKS_CLUSTER_NAME}"
                            // Kubernetes에 배포
                            sh "kubectl apply -f output.yaml"
                            // Kubernetes에 서비스 배포
                            sh "kubectl apply -f kubernetes/service.yaml"
                            // 임시로 생성한 파일 삭제
                            sh "rm output.yaml"
                        }
                    }
                }
            }
        }
    }
}
 
