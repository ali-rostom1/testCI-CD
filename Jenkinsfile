pipeline {
  agent any

  environment {
    GIT_REPO_URL = 'https://github.com/ali-rostom1/testCI-CD'
    DOCKER_CREDENTIALS_ID = 'dockerhub-credentials'
    VALID_SERVICES = 'auth-service,booking-service,payment-service'
  }

  stages {
    stage('Checkout PR Code') {
      steps {
        script {
          // First try standard checkout (works for multibranch pipelines)
          checkout scm
          
          // If no PR variables exist, try manual detection
          if (!env.CHANGE_ID) {
            // Extract PR number from branch name (for GitHub)
            def branchName = env.BRANCH_NAME ?: sh(script: 'git rev-parse --abbrev-ref HEAD', returnStdout: true).trim()
            def prMatch = branchName =~ /PR-(\d+)/
            if (prMatch) {
              env.PR_ID = prMatch[0][1]
              echo "Detected PR #${env.PR_ID} from branch name"
            } else {
              error("Not running as a PR build. Branch '${branchName}' doesn't appear to be a PR branch.")
            }
          } else {
            env.PR_ID = env.CHANGE_ID
            echo "Building PR #${env.PR_ID}"
          }
          
          // Fetch PR-specific refspec
          checkout([
            $class: 'GitSCM',
            branches: [[name: 'FETCH_HEAD']],
            extensions: [
              [$class: 'CleanBeforeCheckout'],
              [$class: 'CloneOption', depth: 1, shallow: true, noTags: true]
            ],
            userRemoteConfigs: [[
              url: env.GIT_REPO_URL,
              refspec: "+refs/pull/${env.PR_ID}/head:refs/remotes/origin/PR-${env.PR_ID}",
              credentialsId: 'github_credentials'
            ]]
          ])
        }
      }
    }

    stage('Detect Changed Services') {
      steps {
        script {
          // Get target branch (default to main)
          def targetBranch = env.CHANGE_TARGET ?: 'main'
          sh "git fetch origin ${targetBranch} --depth=1"
          
          def changes = sh(
            script: """
              git diff --name-only origin/${targetBranch}...HEAD -- \
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
            error("No changes in monitored services. Stopping pipeline.")
          }
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
              sh '''#!/bin/bash
                echo "Building Docker image..."
                docker build -t alirostom220/${AFFECTED_SERVICES}:${prTag} .
                echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                docker push "alirostom220/${AFFECTED_SERVICES}:${prTag}"
                docker logout
              '''
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
          sh '''#!/bin/bash
            curl -sS -X POST \
              -H "Authorization: token $GITHUB_TOKEN" \
              -H "Accept: application/vnd.github.v3+json" \
              "https://api.github.com/repos/ali-rostom1/testCI-CD/issues/${PR_ID}/comments" \
              -d '{"body":"✅ PR Build Successful!\\nServices: ${AFFECTED_SERVICES}\\nImages: alirostom220/${AFFECTED_SERVICES}:pr-${PR_ID}-${BUILD_NUMBER}"}'
          '''
        }
      }
    }
    failure {
      script {
        if (env.PR_ID) {
          withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
            sh '''#!/bin/bash
              curl -sS -X POST \
                -H "Authorization: token $GITHUB_TOKEN" \
                -H "Accept: application/vnd.github.v3+json" \
                "https://api.github.com/repos/ali-rostom1/testCI-CD/issues/${PR_ID}/comments" \
                -d '{"body":"❌ PR Build Failed!\\nCheck logs: ${BUILD_URL}"}'
            '''
          }
        }
      }
    }
  }
}