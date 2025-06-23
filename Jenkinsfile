pipeline {
  agent any

  // Enable GitHub PR triggers
  triggers {
    githubPullRequest(
      useGitHubHooks: true,
      permitAll: false,
      autoCloseFailedPullRequests: false
    )
  }

  environment {
    GIT_REPO_URL = 'https://github.com/ali-rostom1/testCI-CD'
    DOCKER_CREDENTIALS_ID = 'dockerhub-credentials'
    VALID_SERVICES = 'auth-service,booking-service,payment-service'
  }

  stages {
    stage('Verify PR Context') {
      steps {
        script {
          // Debug all environment variables
          sh 'printenv | sort'
          
          // Check for GitHub PR variables
          if (!env.CHANGE_ID && !env.ghprbPullId) {
            error("Not running as a PR build. CHANGE_ID and ghprbPullId are both missing.")
          }
          
          // Set consistent PR ID variable
          env.PR_ID = env.CHANGE_ID ?: env.ghprbPullId
          echo "Building PR #${env.PR_ID}"
        }
      }
    }

    stage('Checkout PR Code') {
      steps {
        script {
          checkout([
            $class: 'GitSCM',
            branches: [[name: 'FETCH_HEAD']],
            extensions: [
              [$class: 'CleanBeforeCheckout'],
              [$class: 'CloneOption',
               depth: 1,
               shallow: true,
               noTags: true]
            ],
            userRemoteConfigs: [[
              url: env.GIT_REPO_URL,
              refspec: "+refs/pull/${env.PR_ID}/head:refs/remotes/origin/PR-${env.PR_ID}",
              credentialsId: 'github_credentials'
            ]]
          ])
          sh 'git show-ref && git log -1'
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