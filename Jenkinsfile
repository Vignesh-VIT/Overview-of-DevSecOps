def COLOR_MAP = [
    'FAILURE' : 'danger',
    'SUCCESS' : 'good'
]

pipeline {
    agent any
    
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    
    environment {
        SCANNER_HOME= tool 'sonar-scanner'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-credentials', url: 'https://github.com/Vignesh-VIT/Boardgame.git'
            }
        }
        stage('Maven Compile') {
            steps {
                bat 'mvn clean install'
            }
        }
        stage('Test') {
            steps {
                bat 'mvn test'
            }
        }
        stage('File System Scan') {
            steps {
                bat 'trivy fs --format table -o trivy-fs-report.html .'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    bat " %SCANNER_HOME%\\bin\\sonar-scanner -X -Dsonar.java.binaries=target/classes -Dsonar.dynamicAnalysis=reuseReports -Dsonar.jacoco.reportPath=${workspace}\\${env.JOB_NAME}\\target\\jacaco.exec -Dsonar.junit.reportPath=${workspace}\\${env.JOB_NAME}\\target\\surefire-reports\\ -Dsonar.projectName=BoardGame -Dsonar.projectKey=BoardGame -Dsonar.language=java  -Dsonar.core.codeCoveragePlugin=jacoco  -Dsonar.sources=src  -Dplugin.tests.sonar.tests=src/test/** "
                    
                }
            }
        }
        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token' 
            }
        }
        stage('Build') {
            steps {
                bat 'mvn package'
            }
        }
        stage('Publish to Nexus Repository') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    bat 'mvn deploy'
                }
            }
        }
        stage('Build & Tag Docker Image') {
            steps {
                bat 'docker build -t vigneshvit/boardgame:%BUILD_ID% .'
                bat 'docker tag vigneshvit/boardgame:%BUILD_ID% vigneshvit/boardgame:latest'
                bat 'docker rmi vigneshvit/boardgame:%BUILD_ID%'
            }
        }
        stage('Docker Image Scan') {
            steps {
                bat 'trivy image --format table -o trivy-image-report.html vigneshvit/boardgame:latest'
            }
        }
        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-credentials', passwordVariable: 'dockerHubPassword', usernameVariable: 'dockerHubUser')]) {
                bat 'docker login -u %dockerHubUser% -p %dockerHubPassword%'
                bat 'docker push vigneshvit/boardgame:latest'
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'docker-desktop', contextName: '', credentialsId: 'k8s-credentials', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://127.0.0.1:60490') {
                        bat 'kubectl apply -f deployment-service.yaml'
                }
            }
        }
        stage('Verify the Deployment Kubernetes') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'docker-desktop', contextName: '', credentialsId: 'k8s-credentials', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://127.0.0.1:60490') {
                        bat 'kubectl get pods -n webapps'
                        bat 'kubectl get svc -n webapps'
                }
            }
        }
    }
    
    post {
        always { 
            echo 'Slack Notifications'
            slackSend (
                channel: '#devops-and-infra-team',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} \n build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
            )
        }
    }
}