pipeline {
    agent any

    environment {
        REGISTRY = 'user07.azurecr.io'
        IMAGE_NAME = 'product'
        AKS_CLUSTER = 'user07-aks'
        RESOURCE_GROUP = 'user07-rsrcgrp'
        AKS_NAMESPACE = 'default'
        AZURE_CREDENTIALS_ID = 'Azure-Cred' // Jenkins에 설정한 Azure 자격 증명 ID
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
                    image = docker.build("${REGISTRY}/${IMAGE_NAME}")
                }
            }
        }
        
        stage('Push to ACR') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: env.AZURE_CREDENTIALS_ID, usernameVariable: 'AZURE_CLIENT_ID', passwordVariable: 'AZURE_CLIENT_SECRET')]) {
                        sh 'az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET --tenant <tenant-id>'
                        sh "az acr login --name ${REGISTRY.split('.')[0]}"
                        sh "docker push ${REGISTRY}/${IMAGE_NAME}:v${env.BUILD_NUMBER}"
                    }
                }
            }
        }
        
        stage('CleanUp Images') {
            steps {
                sh """
                docker rmi ${REGISTRY}/${IMAGE_NAME}:v$BUILD_NUMBER
                """
            }
        }
        
        stage('Deploy to AKS') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: env.AZURE_CREDENTIALS_ID, usernameVariable: 'AZURE_CLIENT_ID', passwordVariable: 'AZURE_CLIENT_SECRET')]) {
                        sh 'az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET --tenant <tenant-id>'
                        sh "az aks get-credentials --resource-group ${RESOURCE_GROUP} --name ${AKS_CLUSTER}"
                        sh """
                        sed 's/latest/v${env.BUILD_ID}/g' kubernetes/deploy.yaml > output.yaml
                        cat output.yaml
                        kubectl apply -f output.yaml
                        kubectl apply -f kubernetes/service.yaml
                        rm output.yaml
                        """
                    }
                }
            }
        }
    }
}
