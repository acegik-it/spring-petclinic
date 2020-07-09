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
      resources:
       requests:
        cpu: "0.05"
        memory: "256Mi"
       limits:
        cpu: "2"
        memory: "2Gi"
    - name: 'dind'
      image: docker:dind
      securityContext:
        privileged: true
    - name: 'kubectl'
      image: lachlanevenson/k8s-kubectl:v1.14.2
      command:
        - cat
      tty: true
"""
        }
    }

    environment {
        VERSION = ""
        APP_NAME = ""
        APP_GROUP = ""
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
                    def pom = readMavenPom()
                    def current_version = pom.getVersion()
                    VERSION = current_version+'.'+BUILD_NUMBER
                    APP_NAME = pom.getArtifactId()
                    APP_GROUP = pom.getGroupId()
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
                    withDockerRegistry([ credentialsId: "ZAN_DOCKER_CREDENTIALS",  url: "" ]) {
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
        }

        stage('Push Image') {
            steps {
                container('dind') {
                    withDockerRegistry([ credentialsId: "ZAN_DOCKER_CREDENTIALS", url: "" ]) {
                        sh "docker push zandolsi/spring-petclnic:${VERSION}"
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                container('kubectl') {
                    sh"""
                        sed -i 's/IMAGE_TAG/${VERSION}/g' deployment.yaml
                        kubectl apply -f deployment.yaml -n jenkins
                    """
                }
            }
        }
    }
}
