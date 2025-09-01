pipeline {
  agent any

  tools {
    // If you added JDK11, keep this; otherwise remove this line and Jenkins will use your system JDK (21 in your log)
    jdk 'JDK11'
    maven 'Maven3'
  }

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
          if (isUnix()) {
            sh "mvn ${env.MAVEN_ARGS} --version"
            sh "mvn ${env.MAVEN_ARGS} clean validate compile -DskipTests"
            sh "mvn ${env.MAVEN_ARGS} test"
          } else {
            bat "mvn ${env.MAVEN_ARGS} --version"
            bat "mvn ${env.MAVEN_ARGS} clean validate compile -DskipTests"
            bat "mvn ${env.MAVEN_ARGS} test"
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
          if (isUnix()) {
            sh "mvn ${env.MAVEN_ARGS} package -DskipTests"
          } else {
            bat "mvn ${env.MAVEN_ARGS} package -DskipTests"
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
