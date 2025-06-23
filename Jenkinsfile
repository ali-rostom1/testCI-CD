pipeline {
  agent any

  environment {
    DOCKER_CREDENTIALS_ID = 'dockerhub-credentials'
    VALID_SERVICES = 'auth-service,booking-service,payment-service'
    // GitHub plugin automatically provides:
    // CHANGE_ID (PR number), CHANGE_TARGET (base branch), CHANGE_BRANCH (PR branch)
    PR_ID = "${env.CHANGE_ID}"
    TARGET_BRANCH = "${env.CHANGE_TARGET ?: 'main'}"
  }

  stages {
    stage('Checkout PR Code') {
      steps {
        checkout([
          $class: 'GitSCM',
          branches: [[name: "refs/pull/${env.PR_ID}/head"]],
          extensions: [
            [$class: 'CleanBeforeCheckout'],
            [$class: 'CloneOption', depth: 1, shallow: true, noTags: true]
          ],
          userRemoteConfigs: [[
            url: 'https://github.com/ali-rostom1/testCI-CD',
            refspec: "+refs/pull/${env.PR_ID}/head:refs/remotes/origin/PR-${env.PR_ID}",
            credentialsId: 'github_credentials'
          ]]
        ])
      }
    }

    stage('Detect Changed Services') {
      steps {
        script {
          sh "git fetch origin ${env.TARGET_BRANCH} --depth=1"
          
          def changes = sh(
            script: """
              git diff --name-only origin/${env.TARGET_BRANCH}...HEAD -- \
              | cut -d/ -f1 \
              | sort -u \
              | grep -vE '^\\.'
            """,
            returnStdout: true
          ).trim()

          def validServices = env.VALID_SERVICES.split(',').collect { it.trim() }
          def affected = changes.split('\n').findAll { it in validServices }

          if (affected.empty) {
            currentBuild.result = 'SUCCESS'
            error("No changes in monitored services. Stopping pipeline.")
          }
          env.AFFECTED_SERVICES = affected.join(',')
          echo "Changes detected in: ${env.AFFECTED_SERVICES}"
        }
      }
    }

    stage('Build and Push') {
      steps {
        script {
          def prTag = "pr-${env.PR_ID}-${env.BUILD_NUMBER}"
          
          dir(env.AFFECTED_SERVICES) {
            withCredentials([usernamePassword(
              credentialsId: env.DOCKER_CREDENTIALS_ID,
              usernameVariable: 'DOCKER_USER',
              passwordVariable: 'DOCKER_PASS'
            )]) {
              sh """
                docker build -t alirostom220/${env.AFFECTED_SERVICES}:${prTag} .
                echo "\$DOCKER_PASS" | docker login -u "\$DOCKER_USER" --password-stdin
                docker push "alirostom220/${env.AFFECTED_SERVICES}:${prTag}"
                docker logout
              """
            }
          }
        }
      }
    }
  }

  post {
    success {
      script {
        withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
          sh """
            curl -sS -X POST \
              -H "Authorization: token \$GITHUB_TOKEN" \
              -H "Accept: application/vnd.github.v3+json" \
              "https://api.github.com/repos/ali-rostom1/testCI-CD/issues/${env.PR_ID}/comments" \
              -d '{"body":"✅ PR Build Successful!\\nServices: ${env.AFFECTED_SERVICES}\\nImages: alirostom220/${env.AFFECTED_SERVICES}:${prTag}"}'
          """
        }
      }
    }
    failure {
      script {
        withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
          sh """
            curl -sS -X POST \
              -H "Authorization: token \$GITHUB_TOKEN" \
              -H "Accept: application/vnd.github.v3+json" \
              "https://api.github.com/repos/ali-rostom1/testCI-CD/issues/${env.PR_ID}/comments" \
              -d '{"body":"❌ PR Build Failed!\\nCheck logs: ${env.BUILD_URL}"}'
          """
        }
      }
    }
  }
}