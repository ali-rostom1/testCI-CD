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
            depth: 0,  // Full clone history
            shallow: false,
            noTags: false
          ]],
          userRemoteConfigs: [[
            url: env.GIT_REPO_URL,
            credentialsId: 'github_credentials' 
          ]]
        ])
        
        // Use the same credentials for subsequent git operations
        withCredentials([usernamePassword(
          credentialsId: 'github_credentials',
          usernameVariable: 'GIT_USERNAME',
          passwordVariable: 'GIT_PASSWORD'
        )]) {
          sh '''
            git config --global url."https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com"
            git fetch origin '+refs/heads/*:refs/remotes/origin/*' --update-head-ok
            git fetch origin '+refs/pull/*:refs/remotes/origin/pr/*' --update-head-ok
          '''
        }
      }
    }

    stage('Detect Changed Services') {
      steps {
        script {
          // Find merge base with target branch (main)
          def mergeBase = sh(
            script: "git merge-base HEAD origin/main",
            returnStdout: true
          ).trim()
          
          echo "Merge base with main: ${mergeBase}"
          
          // Get all changed files since branching from main
          def changes = sh(
            script: "git diff --name-only ${mergeBase}..HEAD | cut -d/ -f1 | sort -u",
            returnStdout: true
          ).trim()
          
          echo "All changed top-level directories: ${changes}"
          
          // Split changes and filter valid services
          def changedList = changes ? changes.split("\n") : []
          def affected = changedList.findAll { service -> 
            env.VALID_SERVICES.split(",").contains(service.trim()) 
          }
          
          if (affected.isEmpty()) {
            currentBuild.result = 'SUCCESS'
            echo "No microservice changes detected. Skipping build."
            return
          }

          env.AFFECTED_SERVICES = affected.join(",")
          echo "Affected services: ${env.AFFECTED_SERVICES}"
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