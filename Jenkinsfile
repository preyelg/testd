pipeline {
  agent any

  tools {
    jdk 'JDK11'
    maven 'Maven3'
  }

  options {
    buildDiscarder(logRotator(numToKeepStr: '20'))
    durabilityHint('PERFORMANCE_OPTIMIZED')
    timeout(time: 20, unit: 'MINUTES')
  }

  environment {
    // Keep Maven quiet and CI-friendly
    MAVEN_OPTS = '-Dorg.slf4j.simpleLogger.showDateTime=true -Djava.awt.headless=true'
    MAVEN_ARGS = '-B -ntp'
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Test') {
      steps {
        script {
          // Prefer wrapper if present; otherwise use system Maven from tools{}
          def mvnCmd = fileExists('mvnw') ? (isUnix() ? './mvnw' : 'mvnw.cmd') : 'mvn'
          if (isUnix()) {
            sh "${mvnCmd} ${env.MAVEN_ARGS} --version"
            sh "${mvnCmd} ${env.MAVEN_ARGS} clean validate compile -DskipTests"
            sh "${mvnCmd} ${env.MAVEN_ARGS} test"
          } else {
            bat "${mvnCmd} ${env.MAVEN_ARGS} --version"
            bat "${mvnCmd} ${env.MAVEN_ARGS} clean validate compile -DskipTests"
            bat "${mvnCmd} ${env.MAVEN_ARGS} test"
          }
        }
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: '**/surefire-reports/*.xml'
        }
      }
    }

    stage('Package WAR') {
      steps {
        script {
          def mvnCmd = fileExists('mvnw') ? (isUnix() ? './mvnw' : 'mvnw.cmd') : 'mvn'
          if (isUnix()) {
            sh "${mvnCmd} ${env.MAVEN_ARGS} package -DskipTests"
          } else {
            bat "${mvnCmd} ${env.MAVEN_ARGS} package -DskipTests"
          }
        }
      }
    }

    stage('Archive Artifact') {
      steps {
        archiveArtifacts artifacts: 'target/*.war', fingerprint: true
      }
    }
  }

  post {
    success { echo 'Build complete. WAR archived.' }
    failure { echo 'Build failed. Check the first error above.' }
    always {
      cleanWs(deleteDirs: true, notFailBuild: true)
    }
  }
}
