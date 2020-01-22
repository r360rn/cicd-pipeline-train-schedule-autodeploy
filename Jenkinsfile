pipeline {
    agent any
    environment {
        DOCKER_IMAGE_NAME = "reb0rn/testing"
        KUBE_NODE_IP = '3.216.27.73'
        TEST_STATE = true
    }
    stages {
        stage('Build') {
            steps {
                echo 'Running build automation'
                sh './gradlew build --no-daemon'
                archiveArtifacts artifacts: 'dist/trainSchedule.zip'
            }
        }
        stage('Build Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    app = docker.build(DOCKER_IMAGE_NAME)
                    app.inside {
                        sh 'echo Hello, World!'
                    }
                }
            }
        }
        stage('Push Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        stage('CanaryDeploy') {
            when {
                branch 'master'
            }
            environment { 
                CANARY_REPLICAS = 1
            }
            steps {
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig',
                    configs: 'train-schedule-kube-canary.yml',
                    enableConfigSubstitution: true
                )
            }
        }
        stage('Smoke Test'){
            when{
                branch 'master'
            }
            steps{
                script{
                    webrequest = httpRequest(
                        url: "http://$KUBE_NODE_IP:8081",
                        timeout: 30
                    )
                    if (webrequest.status != 200){
                        TEST_STATE = false
                    }
                    }
                }
            }
        stage('DeployToProduction') {
            when {
                branch 'master'
            }
            environment { 
                CANARY_REPLICAS = 0
            }
            steps {
                script{
                    if (TEST_STATE) {
                        kubernetesDeploy(
                            kubeconfigId: 'kubeconfig',
                            configs: 'train-schedule-kube-canary.yml',
                            enableConfigSubstitution: true
                        )
                        kubernetesDeploy(
                            kubeconfigId: 'kubeconfig',
                            configs: 'train-schedule-kube.yml',
                            enableConfigSubstitution: true
                        )
                }
                }
            }
        }
    }
}
