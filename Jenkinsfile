pipeline {
    agent any

    tools {
        nodejs "node JS"
    }

    environment {
        github_token = credentials('githubcredentails')
        AWS_REGION = 'us-east-1' 
        CLUSTER_NAME = 'my-ecs-cluster'
        SERVICE_NAME = 'my-ecs-service' 
        TASK_FAMILY  = 'logoswayatt-task' 
        IMAGE_NAME   = "muskaan810/logoswayatt"
    }

    stages {

        stage('Install Dependency') {
            steps {
                sh 'npm install --no-audit'
            }
        }

        stage('Dependency Scanning') {
            parallel {
              stage('npm Dependency Audit') {
                steps {
                  sh 'npm audit --audit-level=critical'
                }
              }
              stage('OWASP Dependency-Check Vulnerabilities') {
                steps {
                  dependencyCheck additionalArguments: '''
                    -o './'
                    -s './'
                    -f 'ALL'
                    --prettyPrint
                  ''', odcInstallation: 'owasp-tool'

                  dependencyCheckPublisher pattern: 'dependency-check-report.xml'
                }
              }
            }
      }

        stage("Build Docker Image") {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${GIT_COMMIT} ."
            }
        }

        stage("Push to DockerHub") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub-credentials', url: 'https://index.docker.io/v1/') {
                        sh "docker push ${IMAGE_NAME}:${GIT_COMMIT}"
                    }
                }
            }
        }

        
    }


    post {
          always {


                                  publishHTML(target: [
                                  allowMissing: false,
                                  alwaysLinkToLastBuild: true,
                                  keepAll: true,
                                  reportDir: './',
                                  reportFiles: 'dependency-check-report.html',
                                  reportName: 'Oswao dependency check report'
                              ]) 
                              cleanWs()
          }
        } 
}  



