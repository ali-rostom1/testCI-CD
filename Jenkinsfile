pipeline {
  agent any

  environment {
    GIT_REPO_URL = 'https://github.com/ali-rostom1/testCI-CD'
    DOCKER_CREDENTIALS_ID = 'dockerhub-credentials'
    VALID_SERVICES = 'auth-service,booking-service,payment-service'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout([
          $class: 'GitSCM',
          branches: [[name: '*/main']],
          extensions: [[
            $class: 'CloneOption',
            depth: 0,
            shallow: false,
            noTags: false
          ]],
          userRemoteConfigs: [[
            url: env.GIT_REPO_URL,
            credentialsId: 'github_credentials'
          ]]
        ])
        
        // Configure Git to use credentials
        withCredentials([usernamePassword(
          credentialsId: 'github_credentials',
          usernameVariable: 'GIT_USER',
          passwordVariable: 'GIT_PASS'
        )]) {
          sh '''
            git config --global url."https://${GIT_USER}:${GIT_PASS}@github.com".insteadOf "https://github.com"
            git fetch origin '+refs/heads/*:refs/remotes/origin/*' --update-head-ok
            git fetch origin '+refs/pull/*:refs/remotes/origin/pr/*' --update-head-ok
          '''
        }
      }
    }

    stage('Detect Changed Services') {
      steps {
        script {
          // Get the previous successful commit on this branch
          def previousCommit = sh(
            script: "git rev-parse HEAD~1",
            returnStdout: true
          ).trim()
          
          // Get current commit
          def currentCommit = sh(
            script: "git rev-parse HEAD",
            returnStdout: true
          ).trim()
          
          echo "Comparing changes between ${previousCommit} and ${currentCommit}"
          
          // Get changed files between commits
          def changes = sh(
            script: "git diff --name-only ${previousCommit} ${currentCommit} | cut -d/ -f1 | sort -u",
            returnStdout: true
          ).trim()
          
          echo "Raw changed directories:\n${changes}"
          
          // Split and filter valid services
          def changedList = changes.split("\n").collect { it.trim() }.findAll { it }
          def affected = changedList.findAll { service ->
            VALID_SERVICES.split(",")*.trim().contains(service)
          }
          
          if (affected.isEmpty()) {
            currentBuild.result = 'SUCCESS'
            echo "No service changes detected in: ${env.VALID_SERVICES}"
            return
          }

          env.AFFECTED_SERVICES = affected.join(",")
          echo "Detected changes in services: ${env.AFFECTED_SERVICES}"
        }
      }
    }

    // Rest of your pipeline remains the same...
    stage('Build, Test & Deploy') {
      when {
        expression { return env.AFFECTED_SERVICES?.trim() }
      }
      steps {
        script {
          def services = env.AFFECTED_SERVICES.split(",")
          for (service in services) {
            dir(service) {
              stage("→ ${service}: Docker Build & Push") {
                withCredentials([usernamePassword(
                  credentialsId: "${DOCKER_CREDENTIALS_ID}",
                  usernameVariable: 'DOCKER_USER',
                  passwordVariable: 'DOCKER_PASS'
                )]) {
                  def tag = "${env.BUILD_NUMBER}"
                  def image = "adtripy/${service}:${tag}"

                  sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                  sh "docker build -t ${image} ."
                  sh "docker push ${image}"
                  sh 'docker logout'
                }
              }
            }
          }
        }
      }
    }
  }

  post {
    success {
      echo "✅ Pipeline succeeded for: ${env.AFFECTED_SERVICES ?: 'No service changes'}"
    }
    failure {
      echo "❌ Pipeline failed. Check logs for details."
    }
    aborted {
      echo "🟨 Pipeline aborted."
    }
  }
}