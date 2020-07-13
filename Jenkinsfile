pipeline {

    agent {
        kubernetes {
            label 'petclinic-app'
            defaultContainer 'jnlp'
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    pipeline: jenkinsfile
spec:
  containers:
    - name: 'maven'
      image: maven:3.3.9
      command:
        - cat
      tty: true
    - name: 'dind'
      image: docker:dind
      securityContext:
        privileged: true
    - name: 'kubectl'
      image: thedevopschris/amazon-eks-kubectl
      command:
        - cat
      tty: true
"""
        }
    }

    environment {
        DOCKER_CREDENTIALS = credentials('ZAN_DOCKER_CREDENTIALS')
        EKS_PREPROD_CONFIG = credentials('EKS_PREPROD_CONFIG')
        EKS_PROD_CONFIG = credentials('EKS_PROD_CONFIG')
        VERSION = ""
    }

    options {
        skipDefaultCheckout(true)
        buildDiscarder(logRotator(numToKeepStr: '5'))
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    deleteDir()
                    checkout scm
                    def matcher = readFile('pom.xml') =~ '<version>(.+?)</version>'
                    def current_version = matcher ? matcher[0][1] : '0.1.0'
                    VERSION = current_version+'.'+BUILD_NUMBER
                }
            }
        }

        stage('Compile') {
            steps {
                container('maven') {
                    script {
                        sh """
                            mvn versions:set -DnewVersion=${VERSION}
                            mvn clean compile
                        """
                    }
                }
            }
        }

        stage('Test') {
            steps {
                container('maven') {
                    sh "mvn verify -DskipTests"
                }
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Build Image') {
            steps {
                container('dind') {
                        sh"""
                            mkdir target/working-dir
                            cp -R src/main/resources/docker/* target/working-dir/
                            cp target/spring-petclinic*.jar target/working-dir/
                            cd target/working-dir
                            docker build -t zandolsi/spring-petclnic:${VERSION} .
                        """
                }
            }
        }

        stage('Push Image') {
            steps {
                container('dind') {
                        sh"""
                        docker login -u ${DOCKER_CREDENTIALS_USR} -p ${DOCKER_CREDENTIALS_PSW}
                        docker push zandolsi/spring-petclnic:${VERSION}
                        """
                }
            }
        }

        stage('Deploy to Preprod') {
            steps {
                withAWS(credentials: 'AWS_CREDENTIALS', region: 'eu-west-2') {
                        container('kubectl') {
                          writeFile file: "$JENKINS_AGENT_WORKDIR/.kube/config", text: readFile(EKS_PREPROD_CONFIG)
                          sh"""
                            export KUBECONFIG=$JENKINS_AGENT_WORKDIR/.kube/config
                            sed -i 's/IMAGE_TAG/${VERSION}/g' deployment.yaml
                            kubectl get pods -n petclinic
                          """
                        }
               }
            }
        }

       stage('E2E Test') {
           steps {
                sh "echo executing e2e tests..."
           }
       }

       stage('Deploy to Preprod') {
            steps {
                withAWS(credentials: 'AWS_CREDENTIALS', region: 'eu-west-2') {
                        container('kubectl') {
                          writeFile file: "$JENKINS_AGENT_WORKDIR/.kube/config", text: readFile(EKS_PROD_CONFIG)
                          sh"""
                            export KUBECONFIG=$JENKINS_AGENT_WORKDIR/.kube/config
                            sed -i 's/IMAGE_TAG/${VERSION}/g' deployment.yaml
                            kubectl get pods -n petclinic
                          """
                        }
                }
            }
        }
    }
}
