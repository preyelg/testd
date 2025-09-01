pipeline {
  agent {
    // Uses a stable Maven+JDK11 container so you don't depend on node config
    docker {
      image 'maven:3.9.6-eclipse-temurin-11'
      // Optional: cache dependencies between builds (uncomment if you want)
      // args '-v $JENKINS_HOME/maven-cache:/root/.m2'
      reuseNode true
    }
  }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
    durabilityHint('PERFORMANCE_OPTIMIZED')
    ansiColor('xterm')
    timeout(time: 20, unit: 'MINUTES')
  }

  environment {
    // Tweak Maven output to be CI-friendly
    MAVEN_OPTS = '-Dorg.slf4j.simpleLogger.showDateTime=true -Djava.awt.headless=true'
    MAVEN_ARGS = '-B -ntp'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Validate & Compile') {
      steps {
        sh "mvn ${env.MAVEN_ARGS} clean validate compile -DskipTests"
      }
    }

    stage('Unit Tests') {
      steps {
        sh "mvn ${env.MAVEN_ARGS} test"
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: '**/surefire-reports/*.xml'
        }
      }
    }

    stage('Package WAR') {
      steps {
        // Reuse compiled classes; don't re-run tests here (they already ran)
        sh "mvn ${env.MAVEN_ARGS} package -DskipTests"
      }
    }

    stage('Archive Artifact') {
      steps {
        // NumberGuessGame WAR lands under target/*.war per pom packaging
        archiveArtifacts artifacts: 'target/*.war', fingerprint: true
      }
    }
  }

  post {
    success {
      echo 'Build complete. WAR archived.'
    }
    failure {
      echo 'Build failed. Check logs above for the first error.'
    }
    always {
      cleanWs(deleteDirs: true, notFailBuild: true)
    }
  }
}
