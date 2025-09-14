pipeline {
  agent any

  tools {
    nodejs "node JS"
  }

 environment {
  github_token =credentials('githubcredentails')
  IMAGE_NAME   = "muskaan810/logoswayatt"
}


  stages {

    
      stage('Install Dependency') {
        options { timestamps () }
        steps {
          sh 'npm install --no-audit'
        }
      }

    // stage('Auto Fix (Safe Upgrades)') {
    //   steps {
    //     sh 'npm audit fix || true'
    //   }
    // }

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

      // stage('Unit Testing') {
      //   options { retry (2) }
      //   steps {
      //      sh 'npm test'
      //   }
      // }
      // stage('Code Coverage') {
      //     steps {
      //             catchError(buildResult: 'SUCCESS', message: 'It will be fixed later', stageResult: 'UNSTABLE') {
      //                 sh 'npm run coverage'
      //           }
      //       }
            
      //   }


      stage (" Building docker image"){
        steps{
          sh "docker build -t $IMAGE_NAME:$GIT_COMMIT ."
        }
      }

      stage("Push to Registry") {
        steps {
          script {
            withDockerRegistry(credentialsId: 'dockerhub-credentials', url: 'https://index.docker.io/v1/') {
              sh "docker push $IMAGE_NAME:$GIT_COMMIT"
            }
          }
        }
      }

      stage("Deploy to ECS") {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                    sh """
                    echo "Registering new ECS task definition..."
                    NEW_TASK_DEF=$(jq -n --arg IMAGE "$IMAGE_NAME:$GIT_COMMIT" '{
                        family: "$TASK_FAMILY",
                        networkMode: "awsvpc",
                        requiresCompatibilities: ["FARGATE"],
                        cpu: "256",
                        memory: "512",
                        containerDefinitions: [{
                            name: "logoswayatt-container",
                            image: $IMAGE,
                            essential: true,
                            portMappings: [{
                                containerPort: 3000,
                                protocol: "tcp"
                            }]
                        }]
                    }')

                    aws ecs register-task-definition \
                        --region $AWS_REGION \
                        --cli-input-json "$NEW_TASK_DEF"

                    echo "Updating ECS service with new task definition..."
                    REVISION=$(aws ecs describe-task-definition \
                        --task-definition $TASK_FAMILY \
                        --query 'taskDefinition.revision' \
                        --output text)

                    aws ecs update-service \
                        --cluster $CLUSTER_NAME \
                        --service $SERVICE_NAME \
                        --task-definition $TASK_FAMILY:\$REVISION \
                        --region $AWS_REGION
                    """
                }
            }
        }
    }
  } 

  post {
      always {
                      // publishHTML(target: [
                      //         allowMissing: false,
                      //         alwaysLinkToLastBuild: true,
                      //         keepAll: true,
                      //         reportDir: 'coverage/lcov-report/',
                      //         reportFiles: 'index.html',
                      //         reportName: 'Code Coverage Html Report'
                      //     ]) 

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
