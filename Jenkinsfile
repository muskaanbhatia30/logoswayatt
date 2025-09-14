pipeline {
    agent any

    tools {
       nodejs "node JS"  // Ensure this matches your NodeJS installation name in Jenkins
    }

    environment {
        github_token = credentials('githubcredentails')
        IMAGE_NAME   = "muskaan810/logoswayatt"
        TASK_FAMILY  = "logoswayatt-task"
        CLUSTER_NAME = "logoswayatt-cluster"
        SERVICE_NAME = "logoswayatt-service"
        AWS_REGION   = "us-east-1"
    }

    stages {

        stage('Install Dependencies') {
            options { timestamps() }
            steps {
                sh 'npm install --no-audit'
            }
        }

        // Optional: auto fix npm vulnerabilities
        // stage('Auto Fix (Safe Upgrades)') {
        //     steps {
        //         sh 'npm audit fix || true'
        //     }
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

        // Optional unit testing and coverage
        // stage('Unit Testing') {
        //     options { retry(2) }
        //     steps {
        //         sh 'npm test'
        //     }
        // }
        // stage('Code Coverage') {
        //     steps {
        //         catchError(buildResult: 'SUCCESS', message: 'It will be fixed later', stageResult: 'UNSTABLE') {
        //             sh 'npm run coverage'
        //         }
        //     }
        // }

        stage("Build Docker Image") {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${GIT_COMMIT} ."
            }
        }

        stage("Push to Docker Registry") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub-credentials', url: 'https://index.docker.io/v1/') {
                        sh "docker push ${IMAGE_NAME}:${GIT_COMMIT}"
                    }
                }
            }
        }

        stage("Deploy to ECS") {
            steps {
                script {
                  withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'Aws-cred']]) {
                    sh """
            echo "Registering new ECS task definition..."
            
            NEW_TASK_DEF='{
                "family": "${TASK_FAMILY}",
                "networkMode": "awsvpc",
                "requiresCompatibilities": ["FARGATE"],
                "cpu": "256",
                "memory": "512",
                "containerDefinitions": [{
                    "name": "logoswayatt-container",
                    "image": "${IMAGE_NAME}:${GIT_COMMIT}",
                    "essential": true,
                    "portMappings": [{
                        "containerPort": 3000,
                        "protocol": "tcp"
                    }]
                }]
            }'

            aws ecs register-task-definition \
                --region ${AWS_REGION} \
                --cli-input-json "\$NEW_TASK_DEF"

            echo "Updating ECS service with new task definition..."
            REVISION=$(aws ecs describe-task-definition \
                --task-definition ${TASK_FAMILY} \
                --query 'taskDefinition.revision' \
                --output text)

            aws ecs update-service \
                --cluster ${CLUSTER_NAME} \
                --service ${SERVICE_NAME} \
                --task-definition ${TASK_FAMILY}:$REVISION \
                --region ${AWS_REGION}
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
                reportName: 'OWASP Dependency Check Report'
            ])
            cleanWs()
        }
    }
}
