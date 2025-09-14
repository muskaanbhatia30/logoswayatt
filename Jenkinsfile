pipeline {
    agent any

    tools {
       nodejs "node JS"  // Make sure this matches your Jenkins NodeJS tool name
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

        stage('Dependency Scanning') {
            parallel(
                "npm Dependency Audit": {
                    steps {
                        sh 'npm audit --audit-level=critical'
                    }
                },
                "OWASP Dependency-Check Vulnerabilities": {
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
            )
        }

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
