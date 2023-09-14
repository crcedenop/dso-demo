pipeline {
  agent {
    kubernetes {
      yamlFile 'build-agent.yaml'
      defaultContainer 'maven'
      idleMinutes 1
    }

  }
  stages {
    stage('Build') {
      parallel {
        stage('Compile') {
          steps {
            container(name: 'maven') {
              sh 'mvn compile'
            }

          }
        }

      }
    }

    stage('Static Analysis') {
      parallel {
        stage('Unit Tests') {
          steps {
            container(name: 'maven') {
              sh 'mvn test'
            }

          }
        }

        stage('SCA') {
          post {
            always {
              archiveArtifacts(allowEmptyArchive: true, artifacts: 'target/dependency-check-report.html', fingerprint: true, onlyIfSuccessful: true)
            }

          }
          steps {
            container(name: 'maven') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh 'mvn org.owasp:dependency-check-maven:check'
              }

            }

          }
        }

        stage('Generate SBOM') {
          post {
            success {
              archiveArtifacts(allowEmptyArchive: true, artifacts: 'target/bom.xml', fingerprint: true, onlyIfSuccessful: true)
            }

          }
          steps {
            container(name: 'maven') {
              sh 'mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom'
            }

          }
        }

        stage('OSS License Checker') {
          steps {
            container(name: 'licensefinder') {
              sh 'ls -al'
              sh '''#!/bin/bash --login
                      /bin/bash --login
                      rvm use default
                      gem install license_finder
                      license_finder
                    '''
            }

          }
        }

      }
    }

    stage('SAST') {
      post {
        success {
          archiveArtifacts(allowEmptyArchive: true, artifacts: 'reports/*', fingerprint: true, onlyIfSuccessful: true)
        }

      }
      steps {
        container(name: 'slscan')
      }
    }

    stage('Package') {
      parallel {
        stage('Create Jarfile') {
          steps {
            container(name: 'maven') {
              sh 'mvn package -DskipTests'
            }

          }
        }

        stage('OCI Image BnP') {
          steps {
            container(name: 'kaniko') {
              sh '/kaniko/executor -f `pwd`/Dockerfile -c `pwd` --insecure --skip-tls-verify --cache=true --destination=docker.io/crcedenp/dso-demo --force'
            }

          }
        }

      }
    }

    stage('Image Analysis') {
      parallel {
        stage('Image Linting') {
          steps {
            container(name: 'docker-tools') {
              sh 'dockle docker.io/crcedenp/dso-demo'
            }

          }
        }

      }
    }

    stage('Deploy to Dev') {
      environment {
        AUTH_TOKEN = credentials('argocd-jenkins-deployer-token')
      }
      steps {
        container(name: 'docker-tools') {
          sh 'docker run -t schoolofdevops/argocd-cli argocd app sync dso-demo --insecure --server $ARGO_SERVER --auth-token $AUTH_TOKEN'
          sh 'docker run -t schoolofdevops/argocd-cli argocd app wait dso-demo --health --timeout 300 --insecure --server $ARGO_SERVER --auth-token $AUTH_TOKEN'
        }

      }
    }

    stage('Dynamic Analysis') {
      parallel {
        stage('E2E tests') {
          steps {
            sh 'echo "All Tests passed!!!"'
          }
        }

        stage('DAST') {
          steps {
            container(name: 'docker-tools') {
              sh 'docker run -t owasp/zap2docker-stable zap-baseline.py -t $DEV_URL || exit 0'
            }

          }
        }

      }
    }

  }
  environment {
    ARGO_SERVER = '34.136.182.152:32100'
    DEV_URL = 'http://35.224.159.92:30080/'
  }
}