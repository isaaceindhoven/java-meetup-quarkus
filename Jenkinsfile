pipeline {
  agent any

  options {
    disableConcurrentBuilds()
    ansiColor('xterm')
  }
  
  environment {
    JAVA_TOOL_OPTIONS="-Djansi.force=true -Duser.home=${WORKSPACE}"
    GIT_SHA_SHORT="${env.GIT_COMMIT.take(7)}"
    APP_NAME="java-meetup-quarkus"
    APP_IMAGE="jwnmulder/${env.APP_NAME}:1.0-${env.GIT_BRANCH.replace('/', '-')}.${env.GIT_SHA_SHORT}"
    KUBE_NAMESPACE="java-meetup"
    KUBE_NAME_PREFIX="${env.GIT_BRANCH.replace('/', '-')}"
    KUBE_DEPLOY_NAME="${env.KUBE_NAME_PREFIX}-${env.APP_NAME}"
  }

  stages {

    stage ('checkout') {
      steps {
        checkout scm
      }
    }

    stage ('build') {
      agent {
        docker reuseNode: true, image: "maven:3.6.2-jdk-11", args: "-e MAVEN_CONFIG=${WORKSPACE}/.m2"
      }
      steps {
        sh "mvn clean package"
        junit '**/target/surefire-reports/*.xml'
      }
    }

    stage ('native build') {
      when { branch 'native' }
      agent {
        docker reuseNode: true, image: "quay.io/quarkus/ubi-quarkus-native-image:19.2.0.1", args: "--entrypoint='' --memory=3g"
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

    stage('prepare deploy') {
      agent {
        docker reuseNode: true, image: "place1/kube-tools:2019.10.13", args: "--entrypoint=''"
      }
      steps {
        dir('deploy') {
            sh """
              > kustomization.yaml
              kustomize edit add base overlays/jenkins-with-nodeport
              kustomize edit set nameprefix '${env.KUBE_NAME_PREFIX}-'
              kustomize edit add label 'app.kubernetes.io/part-of:${env.KUBE_DEPLOY_NAME}'
              kustomize edit set image '${env.APP_IMAGE}'
              kustomize build > generated.yaml
            """
        }
      }
    }

    stage('deploy to test') {
      agent {
        docker reuseNode: true, image: "place1/kube-tools:2019.10.13", args: "--entrypoint=''"
      }
      steps {
        withCredentials([file(credentialsId: "kubectl-config", variable: 'KUBECONFIG')]) {
          sh "kubectl -n ${env.KUBE_NAMESPACE} apply -f deploy/generated.yaml --prune -l 'app.kubernetes.io/part-of=${env.KUBE_DEPLOY_NAME}'"
          sh "kubectl -n ${env.KUBE_NAMESPACE} rollout status  --watch deployment ${env.KUBE_DEPLOY_NAME}"
          sh "kubectl -n ${env.KUBE_NAMESPACE} get deployments,ingress,service -o wide"
          sh "kubectl -n ${env.KUBE_NAMESPACE} logs deployment/${KUBE_DEPLOY_NAME} -c web"

          addTestUrlBadge()
        }
      }
    }

    stage('Testing') {
      parallel {
        stage('automated testing') {
          steps {
            script {
              sh "echo test"
              sleep 60
            }
          }
        }
        stage('manual testing') {
          steps {
            catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
              timeout(time: 15, unit: 'MINUTES')  {
                //input message: "Check results on ${env.APP_TEST_URL}", parameters: [choice(name: 'Tests OK?', choices: ['NOT OK', 'OK'])], submitterParameter: 'manualTestResult'
                input id: 'done-testing', message: 'Done testing?'
              }
            }
          }
        }
      }
    }

    stage('undeploy from test') {
      agent {
        docker reuseNode: true, image: 'place1/kube-tools:2019.10.13', args: "--entrypoint=''"
      }
      steps {
        withCredentials([file(credentialsId: "kubectl-config", variable: 'KUBECONFIG')]) {
          sh "kubectl -n ${env.KUBE_NAMESPACE} delete -f deploy/generated.yaml"
        }

        disableTestUrlBadge()
      }
    }
  }
}

def addTestUrlBadge() {
  script {
    def serviceNodePort = sh(script: "kubectl -n ${env.KUBE_NAMESPACE} get service ${env.KUBE_DEPLOY_NAME}-external -o=jsonpath='{.spec.ports[?(@.port==8080)].nodePort}'", returnStdout: true)
    env.APP_TEST_URL = "http://localhost:${serviceNodePort}/"
  }
  addHtmlBadge id: 'test-url', html: "Test URL: <a href='${env.APP_TEST_URL}'>${env.APP_TEST_URL}</a>"
}

def disableTestUrlBadge() {
  removeHtmlBadges id: 'test-url'
  addHtmlBadge id: 'test-url', html: "Test URL: <a href='${env.APP_TEST_URL}' style='text-decoration: line-through;'>${env.APP_TEST_URL}</a>"
}