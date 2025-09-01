pipeline {
  agent any

  options {
    buildDiscarder(logRotator(numToKeepStr: '20'))
    durabilityHint('PERFORMANCE_OPTIMIZED')
    timeout(time: 20, unit: 'MINUTES')
  }

  environment {
    MAVEN_ARGS = '-B -ntp'
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Test') {
      steps {
        script {
          def mvnCmd = isUnix() ? './mvnw' : 'mvnw.cmd'
          if (isUnix()) {
            sh "java -version || true"
            sh "${mvnCmd} ${env.MAVEN_ARGS} --version"
            sh "${mvnCmd} ${env.MAVEN_ARGS} clean validate compile -DskipTests"
            sh "${mvnCmd} ${env.MAVEN_ARGS} test"
          } else {
            bat "java -version"
            bat "${mvnCmd} ${env.MAVEN_ARGS} --version"
            bat "${mvnCmd} ${env.MAVEN_ARGS} clean validate compile -DskipTests"
            bat "${mvnCmd} ${env.MAVEN_ARGS} test"
          }
        }
      }
      post {
        always { junit allowEmptyResults: true, testResults: '**/surefire-reports/*.xml' }
      }
    }

    stage('Package WAR') {
      steps {
        script {
          def mvnCmd = isUnix() ? './mvnw' : 'mvnw.cmd'
          if (isUnix()) {
            sh "${mvnCmd} ${env.MAVEN_ARGS} package -DskipTests"
          } else {
            bat "${mvnCmd} ${env.MAVEN_ARGS} package -DskipTests"
          }
        }
      }
    }

    stage('Archive Artifact') {
      steps { archiveArtifacts artifacts: 'target/*.war', fingerprint: true }
    }
  }

  post {
    always { cleanWs(deleteDirs: true, notFailBuild: true) }
  }
}
