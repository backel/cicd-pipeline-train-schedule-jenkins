pipeline {
    agent any
    environment {
        DOCKER_IMAGE_NAME="backel/train-schedule"
        CANARY_REPLICAS = 0
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
                        //sh 'echo Hello, World!'
                        sh 'echo $(curl localhost:3000)'
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
                    docker.withRegistry('https://registry.hub.docker.com', 'dockerhub_login') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        stage('Canary Deploy') {
            when {
                branch 'master'
            }
            environment {
                CANARY_REPLICAS = 1
            }
            steps {
                script {
                    withKubeConfig([credentialsId: 'kubeconfig']) {
                        sh "sed -i 's|CANARY_REPLICAS|${CANARY_REPLICAS}|' train-schedule-kube-canary.yml"
                        sh "sed -i 's|DOCKER_IMAGE_NAME|${DOCKER_IMAGE_NAME}|' train-schedule-kube-canary.yml"
                        sh """sed -i "s|BUILD_NUMBER|${env.BUILD_NUMBER}|" train-schedule-kube-canary.yml"""
                        sh "kubectl apply -f train-schedule-kube-canary.yml"
                    }
                }
            }
        }
        stage('Restore .yml file from canary deployment') {
            when {
                branch 'master'
            }
            steps {
                script {
                    sh "git restore train-schedule-kube-canary.yml"
                }
            }
        }
        stage('Smoke Test') {
            when {
                branch 'master'
            }
            steps {
                script {
                    sleep (time: 5)
                    def response = httpRequest (
                        url: "http://$KUBE_MASTER_IP:30010/",
                        timeout: 30
                    )
                    if (response.status != 200) {
                        error("Smoke test against canary deployment failed.")
                    }
                }
            }
        }
        stage('Deploy To Production') {
            when {
                branch 'master'
            }
            steps {
                milestone(1)
                script {
                    withKubeConfig([credentialsId: 'kubeconfig']) {
                        sh "sed -i 's|DOCKER_IMAGE_NAME|${DOCKER_IMAGE_NAME}|' train-schedule-kube.yml"
                        sh """sed -i "s|BUILD_NUMBER|${env.BUILD_NUMBER}|" train-schedule-kube.yml"""
                        sh "kubectl apply -f train-schedule-kube.yml"
                    }
                }
            }
        }
    }
    post {
        cleanup {
            withKubeConfig([credentialsId: 'kubeconfig']) {
                sh "sed -i 's|CANARY_REPLICAS|${CANARY_REPLICAS}|' train-schedule-kube-canary.yml"
                sh "sed -i 's|DOCKER_IMAGE_NAME|${DOCKER_IMAGE_NAME}|' train-schedule-kube-canary.yml"
                sh """sed -i "s|BUILD_NUMBER|${env.BUILD_NUMBER}|" train-schedule-kube-canary.yml"""
                sh "kubectl apply -f train-schedule-kube-canary.yml"
            }
        }
    }
}
