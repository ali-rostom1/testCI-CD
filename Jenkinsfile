pipeline {
  agent any

  environment {
    GIT_REPO_URL = 'https://github.com/ali-rostom1/testCI-CD'
    DOCKER_CREDENTIALS_ID = 'dockerhub-credentials'
    VALID_SERVICES = 'auth-service,booking-service,payment-service'
    // PR-specific variables (properly quoted)
    CHANGE_ID = "${env.CHANGE_ID ?: ''}"  // PR number from GitHub
    CHANGE_TARGET = "${env.CHANGE_TARGET ?: 'main'}"  // Target branch
  }

  stages {
    stage('Check Docker Setup') {
      steps {
        script {
          // Verify Docker is installed and accessible
          def dockerAvailable = sh(
            script: 'which docker || true',
            returnStdout: true
          ).trim()
          
          if (!dockerAvailable) {
            error("Docker not found! Please install Docker on the Jenkins server and add the Jenkins user to the docker group.")
          }
          
          // Verify docker can run
          sh 'docker --version'
        }
      }
    }

    stage('Checkout PR') {
      steps {
        checkout([
          $class: 'GitSCM',
          branches: [[name: 'PR-${CHANGE_ID}']],
          extensions: [[
            $class: 'CloneOption',
            depth: 1,  // Shallow clone for PRs
            shallow: true,
            noTags: true,
            honorRefspec: true
          ]],
          userRemoteConfigs: [[
            url: env.GIT_REPO_URL,
            refspec: '+refs/pull/${CHANGE_ID}/head:refs/remotes/origin/PR-${CHANGE_ID}',
            credentialsId: 'github_credentials'
          ]]
        ])
      }
    }

    stage('Detect PR Changes') {
      steps {
        script {
          // Compare PR branch with target branch
          def changes = sh(
            script: "git diff --name-only origin/${CHANGE_TARGET}...HEAD | cut -d/ -f1 | sort -u",
            returnStdout: true
          ).trim()

          def validServices = env.VALID_SERVICES.split(',').collect { it.trim() }
          def changedList = changes.split('\n').collect { it.trim() }.findAll { it }
          def affected = changedList.findAll { validServices.contains(it) }

          if (affected.isEmpty()) {
            currentBuild.result = 'SUCCESS'
            echo "No changes in monitored services. Skipping build."
            return
          }
          env.AFFECTED_SERVICES = affected.join(',')
          echo "Detected changes in services: ${env.AFFECTED_SERVICES}"
        }
      }
    }

    stage('Build & Test PR') {
      when {
        allOf {
          expression { return env.AFFECTED_SERVICES?.trim() }
          expression { return env.CHANGE_ID }  // Only run for PRs
        }
      }
      steps {
        script {
          def services = env.AFFECTED_SERVICES.split(',')
          for (service in services) {
            dir(service) {
              stage("PR Build → ${service}") {
                // Build with PR tag
                def prTag = "pr-${env.CHANGE_ID}-${env.BUILD_NUMBER}"
                sh "docker build -t alirostom220/${service}:${prTag} ."
                
                // Optional test command (uncomment if needed)
                // sh "docker run --rm alirostom220/${service}:${prTag} ./run-tests.sh"
              }
            }
          }
        }
      }
    }

    stage('Deploy Preview') {
      when {
        allOf {
          expression { return env.AFFECTED_SERVICES?.trim() }
          expression { return env.CHANGE_ID }
          branch 'main'  // Only deploy previews from main PRs
        }
      }
      steps {
        script {
          withCredentials([usernamePassword(
            credentialsId: env.DOCKER_CREDENTIALS_ID,
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASS'
          )]) {
            sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
            
            def services = env.AFFECTED_SERVICES.split(',')
            for (service in services) {
              dir(service) {
                def prTag = "pr-${env.CHANGE_ID}-${env.BUILD_NUMBER}"
                def image = "alirostom220/${service}:${prTag}"
                
                sh """
                  docker push ${image}
                  echo "PR Preview deployed: ${image}"
                """
              }
            }
            sh 'docker logout'
          }
        }
      }
    }
  }

  post {
    success {
      script {
        if (env.CHANGE_ID) {
          // Comment on PR with build results (requires github-token credential)
          withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
            sh """
              curl -sS -X POST \
                -H "Authorization: token \$GITHUB_TOKEN" \
                -H "Accept: application/vnd.github.v3+json" \
                "https://api.github.com/repos/ali-rostom1/testCI-CD/issues/${CHANGE_ID}/comments" \
                -d '{"body":"✅ PR Build Successful!\\nAffected services: ${env.AFFECTED_SERVICES}\\nPreview images pushed with tag: pr-${CHANGE_ID}-${env.BUILD_NUMBER}"}'
            """
          }
        }
      }
    }
    failure {
      echo "❌ Pipeline failed. Check logs for details."
    }
  }
}