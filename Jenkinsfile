pipeline {
  agent any

  tools {
    nodejs "node JS"
  }

 environment {
  github_token =credentials('githubcredentails')
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
          sh "docker build -t muskaan810/logoswayatt:$GIT_COMMIT ."
        }
      }

      stage("Push to Registry") {
        steps {
          script {
            withDockerRegistry(credentialsId: 'dockerhub-credentials', url: 'https://index.docker.io/v1/') {
              sh "docker push muskaan810/logoswayatt:$GIT_COMMIT"
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
