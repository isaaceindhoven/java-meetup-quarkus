pipeline {
  agent any

  options {
    disableConcurrentBuilds()
    ansiColor('xterm')
  }
  
  environment {
    JAVA_TOOL_OPTIONS="-Djansi.force=true -Duser.home=${WORKSPACE}"
    GIT_SHA_SHORT=sh(script: "git rev-parse --short ${GIT_COMMIT}", returnStdout: true).trim()
    APP_IMAGE="jwnmulder/java-meetup-quarkus:1.0-b${env.BUILD_ID}.${env.GIT_SHA_SHORT}"
  }

  stages {
    stage ('checkout') {
      steps {
        checkout scm
      }
    }
    stage ('build') {
      agent {
        docker {
          image "maven:3.6.2-jdk-11"
          reuseNode true
          args "-e MAVEN_CONFIG=${WORKSPACE}/.m2"
        }
      }
      steps {
        sh "mvn clean package"
      }
    }

    stage ('native build') {
      when { branch 'master' }
      agent {
        docker {
          image "quay.io/quarkus/ubi-quarkus-native-image:19.2.0.1"
          reuseNode true
          args "--entrypoint='' --memory=3g"
        }
      }
      steps {
        sh "./mvnw package -Pnative"
        archiveArtifacts 'target/*-runner'
      }
    }

    stage('docker build') {
        steps {
          script {
            def image = docker.build(env.APP_IMAGE, "-f src/main/docker/Dockerfile.jvm --pull .")
            docker.withRegistry("https://registry.hub.docker.com", "docker-hub") {
              image.push()
            }
          }
        }
    }

    stage('report') {
      steps {
        junit '**/target/surefire-reports/*.xml'
      }
    }
  }
}
