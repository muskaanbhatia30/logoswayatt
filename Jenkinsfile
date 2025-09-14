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

        // adding stage to deploy it to ecs
        stage("Deploy to ECS") {
                      steps {
                          withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'Aws-cred']]) {
                              script {
                                  sh """
                                  echo "Creating ECS task definition JSON..."
                                  cat <<EOF > taskdef.json
                  {
                    "family": "${TASK_FAMILY}",
                    "networkMode": "awsvpc",
                    "requiresCompatibilities": ["FARGATE"],
                    "cpu": "256",
                    "memory": "512",
                    "containerDefinitions": [
                      {
                        "name": "logoswayatt-container",
                        "image": "${IMAGE_NAME}:${GIT_COMMIT}",
                        "essential": true,
                        "portMappings": [
                          {
                            "containerPort": 3000,
                            "protocol": "tcp"
                          }
                        ]
                      }
                    ]
                  }
                  EOF

                echo "Registering ECS task definition..."
                aws ecs register-task-definition --cli-input-json file://taskdef.json --region $AWS_REGION

                REVISION=$(aws ecs describe-task-definition --task-definition $TASK_FAMILY --query 'taskDefinition.revision' --output text)

                echo "Updating ECS service to task revision: $REVISION"
                aws ecs update-service --cluster $CLUSTER_NAME --service $SERVICE_NAME --task-definition ${TASK_FAMILY}:\$REVISION --region $AWS_REGION
                """
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



