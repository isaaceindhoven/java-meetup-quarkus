pipeline {
  agent any

  options {
    disableConcurrentBuilds()
    ansiColor('xterm')
  }
  
  environment {
    JAVA_TOOL_OPTIONS="-Djansi.force=true -Duser.home=${WORKSPACE}"
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
      agent {
        docker {
          image "quay.io/quarkus/ubi-quarkus-native-image:19.2.0.1"
          reuseNode true
          args "--entrypoint=''"
        }
      }
      steps {
        sh "./mvnw package -Pnative"
      }
    }

    stage('report') {
      steps {
        junit '**/target/surefire-reports/*.xml'
        archiveArtifacts 'target/*-runner'
      }
    }
  }
}
