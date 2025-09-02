pipeline {
  agent any

  // Use Jenkins-managed tools (recommended). If you didn't add JDK/Maven in Global Tool Config,
  // either add them (names must match below) or remove this "tools" block and rely on system PATH.
  tools {
    jdk   'JDK11'     // or your configured name; OK to compile with Java 21 too
    maven 'Maven3'    // Jenkins will auto-install 3.9.x
  }

  // Handy knobs you can change from the job UI at build time
  parameters {
    booleanParam(name: 'DEPLOY', defaultValue: true, description: 'Auto-deploy to Tomcat after a successful build')
    string(name: 'APP_CONTEXT', defaultValue: 'numberguess', description: 'Context name for the app (URL will be /<context>)')
    string(name: 'TOMCAT_HOME', defaultValue: 'C:\\apache-tomcat-9.0.xx', description: 'Tomcat root (Windows example). On Linux use /opt/tomcat')
    string(name: 'TOMCAT_SERVICE_NAME', defaultValue: '', description: 'Optional Windows service name (e.g., Tomcat9). Leave blank to use startup/shutdown scripts.')
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
        always {
          junit allowEmptyResults: true, testResults: '**/surefire-reports/*.xml'
        }
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
      steps {
        archiveArtifacts artifacts: 'target/*.war', fingerprint: true
      }
    }

    stage('Deploy to Tomcat') {
      when { expression { return params.DEPLOY } }
      steps {
        script {
          if (isUnix()) {
            sh '''
              set -e
              WAR="$(ls -1 target/*.war | head -n1)"
              echo "WAR: $WAR"
              if [ ! -d "$TOMCAT_HOME/webapps" ]; then
                echo "Tomcat webapps not found at $TOMCAT_HOME/webapps"; exit 1
              fi

              # Remove exploded app dir if present
              rm -rf "$TOMCAT_HOME/webapps/${APP_CONTEXT}"

              # Copy WAR to a stable context name
              cp -f "$WAR" "$TOMCAT_HOME/webapps/${APP_CONTEXT}.war"

              # Try service first (systemd), else use scripts
              if command -v systemctl >/dev/null 2>&1; then
                sudo systemctl stop tomcat || true
                sleep 3
                sudo systemctl start tomcat || true
              else
                "$TOMCAT_HOME/bin/shutdown.sh" || true
                sleep 3
                "$TOMCAT_HOME/bin/startup.sh"
              fi
            '''
          } else {
            bat '''
              @echo on
              for %%f in (target\\*.war) do set WAR=%%f
              echo WAR: %WAR%

              if not exist "%TOMCAT_HOME%\\webapps" (
                echo Tomcat webapps not found at %TOMCAT_HOME%\\webapps
                exit /b 1
              )

              rem Remove exploded directory to force clean redeploy
              if exist "%TOMCAT_HOME%\\webapps\\%APP_CONTEXT%" rmdir /s /q "%TOMCAT_HOME%\\webapps\\%APP_CONTEXT%"

              rem Copy WAR under a stable name
              copy /y "%WAR%" "%TOMCAT_HOME%\\webapps\\%APP_CONTEXT%.war"

              rem Restart Tomcat: prefer Windows service if provided; else fallback to scripts
              if not "%TOMCAT_SERVICE_NAME%"=="" (
                sc stop "%TOMCAT_SERVICE_NAME%" || echo Service stop skipped
                timeout /t 3 /nobreak >nul
                sc start "%TOMCAT_SERVICE_NAME%" || echo Service start skipped
              ) else (
                call "%TOMCAT_HOME%\\bin\\shutdown.bat" || echo Shutdown skipped
                timeout /t 3 /nobreak >nul
                call "%TOMCAT_HOME%\\bin\\startup.bat"
              )
            '''
          }
        }
      }
    }
  }

  post {
    success {
      echo "Deployed. Try: http://localhost:8080/${params.APP_CONTEXT}/  (or your Tomcat port)"
    }
    always {
      cleanWs(deleteDirs: true, notFailBuild: true)
    }
  }
}
