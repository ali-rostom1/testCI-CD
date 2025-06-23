pipeline {
    agent any

    // Environment variables for GitHub webhook data (if available)
    environment {
        GITHUB_EVENT = "${env.GITHUB_EVENT_NAME ?: 'manual'}"  // 'pull_request', 'push', etc.
        GITHUB_REF = "${env.GITHUB_REF ?: 'none'}"             // e.g., 'refs/pull/1/merge'
        GITHUB_ACTION = "${env.GITHUB_ACTION ?: 'none'}"        // e.g., 'closed', 'merged'
    }

    stages {
        stage('Checkout') {
            when {
                // Only run if this is a PR merge (GitHub sends a 'push' event on merge)
                expression { 
                    return (env.GITHUB_EVENT == 'push' && env.GITHUB_REF == 'refs/heads/main') 
                    || (env.GITHUB_EVENT == 'pull_request' && env.GITHUB_ACTION == 'closed')
                }
            }
            steps {
                checkout scm
            }
        }

        stage('Build & Test') {
            when {
                expression { 
                    return (env.GITHUB_EVENT == 'push' && env.GITHUB_REF == 'refs/heads/main') 
                    || (env.GITHUB_EVENT == 'pull_request' && env.GITHUB_ACTION == 'closed')
                }
            }
            steps {
                sh 'mvn clean install'  // Replace with your build command
            }
        }

        stage('Deploy to Staging') {
            when {
                // Only deploy if merged into 'main'
                branch 'main'
            }
            steps {
                sh './deploy-to-staging.sh'  // Replace with your deployment script
            }
        }
    }

    post {
        success {
            slackSend channel: '#dev-team', 
                      message: "✅ PR Merged to Main: ${env.JOB_NAME} Build #${env.BUILD_NUMBER} succeeded"
        }
        failure {
            slackSend channel: '#dev-team', 
                      message: "❌ PR Merged to Main: ${env.JOB_NAME} Build #${env.BUILD_NUMBER} failed"
        }
    }
}