pipeline {
  agent any

  environment {
    GIT_REPO_URL = 'https://github.com/ali-rostom1/testCI-CD'
    DOCKER_CREDENTIALS_ID = 'dockerhub-credentials'
    VALID_SERVICES = 'auth-service,booking-service,payment-service'
    // PR-specific variables
    CHANGE_ID = "${env.CHANGE_ID ?: '0'}"  // Default to '0' for non-PR runs
    CHANGE_TARGET = "${env.CHANGE_TARGET ?: 'main'}"
  }

  stages {
    stage('Check Docker Setup') {
      steps {
        script {
          def dockerAvailable = sh(
            script: 'command -v docker || true',
            returnStdout: true
          ).trim()
          
          if (!dockerAvailable) {
            error("Docker not found! Ensure Docker is installed and the Jenkins user has permissions.")
          }
          sh 'docker --version && docker info'
        }
      }
    }

    stage('Checkout PR') {
      steps {
        script {
          // Fail early if not a PR
          if (env.CHANGE_ID == '0') {
            error("Not a pull request. This pipeline requires PR triggers.")
          }

          checkout([
            $class: 'GitSCM',
            branches: [[name: 'FETCH_HEAD']],  // More reliable than pull/ID/head
            extensions: [
              [$class: 'CleanBeforeCheckout'],
              [$class: 'CloneOption',
               depth: 1,
               shallow: true,
               noTags: true,
               honorRefspec: true]
            ],
            userRemoteConfigs: [[
              url: env.GIT_REPO_URL,
              refspec: "+refs/pull/${env.CHANGE_ID}/head:refs/remotes/origin/PR-${env.CHANGE_ID}",
              credentialsId: 'github_credentials'
            ]]
          ])
          
          // Verify PR commit is checked out
          sh 'git log -1 --oneline'
        }
      }
    }

    stage('Detect Changes') {
      steps {
        script {
          // Get target branch HEAD
          sh "git fetch origin ${CHANGE_TARGET} --depth=1"
          
          def changes = sh(
            script: """
              git diff --name-only origin/${CHANGE_TARGET}...HEAD -- \
              | cut -d/ -f1 \
              | sort -u \
              | grep -vE '^\\.'
            """,
            returnStdout: true
          ).trim()

          def validServices = env.VALID_SERVICES.split(',').collect { it.trim() }
          def affected = changes.split('\n').findAll { it in validServices }

          if (affected) {
            env.AFFECTED_SERVICES = affected.join(',')
            echo "Changes detected in: ${env.AFFECTED_SERVICES}"
          } else {
            currentBuild.result = 'SUCCESS'
            echo "No relevant service changes detected."
            return
          }
        }
      }
    }

    stage('Build PR Images') {
      when {
        expression { return env.AFFECTED_SERVICES }
      }
      steps {
        script {
          def prTag = "pr-${env.CHANGE_ID}-${env.BUILD_NUMBER}"
          
          dir(env.AFFECTED_SERVICES) {
            withCredentials([usernamePassword(
              credentialsId: env.DOCKER_CREDENTIALS_ID,
              usernameVariable: 'DOCKER_USER',
              passwordVariable: 'DOCKER_PASS'
            )]) {
              sh """
                docker build -t alirostom220/${env.AFFECTED_SERVICES}:${prTag} .
                echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
                docker push alirostom220/${env.AFFECTED_SERVICES}:${prTag}
                docker logout
              """
            }
          }
        }
      }
    }
  }

  post {
    always {
      script {
        if (env.CHANGE_ID != '0') {
          def status = currentBuild.result == 'SUCCESS' ? 'success' : 'failure'
          withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
            sh """
              curl -sS -X POST \
                -H "Authorization: token \$GITHUB_TOKEN" \
                -H "Accept: application/vnd.github.v3+json" \
                "https://api.github.com/repos/ali-rostom1/testCI-CD/statuses/$(git rev-parse HEAD)" \
                -d '{
                  "state": "${status}",
                  "target_url": "${env.BUILD_URL}",
                  "description": "Jenkins CI",
                  "context": "ci/jenkins"
                }'
            """
          }
        }
      }
    }
  }
}