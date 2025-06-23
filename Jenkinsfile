pipeline {
  agent any

  environment {
    GIT_REPO_URL = 'https://github.com/ali-rostom1/testCI-CD'
    DOCKER_CREDENTIALS_ID = 'dockerhub-credentials'
    //KUBECONFIG_CREDENTIALS_ID = 'kubeconfig-creds'
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
            depth: 0,  // Full clone history for proper diff
            shallow: false
          ]],
          userRemoteConfigs: [[
            url: 'https://github.com/ali-rostom1/testCI-CD.git',
            credentialsId: 'github_credentials'
          ]]
        ])
        
        // Ensure we have the full remote reference
        sh 'git fetch origin'
      }
    }

    stage('Detect Changed Services') {
      steps {
        script {
          // Get the commit hash of the previous build on main
          def previousCommit = sh(
            script: "git rev-parse origin/main~1",
            returnStdout: true
          ).trim()
          
          // Get current commit hash
          def currentCommit = sh(
            script: "git rev-parse HEAD",
            returnStdout: true
          ).trim()
          
          echo "Comparing changes between ${previousCommit} and ${currentCommit}"
          
          // Get changed files between previous and current commit
          def changes = sh(
            script: "git diff --name-only ${previousCommit} ${currentCommit} | cut -d/ -f1 | sort -u",
            returnStdout: true
          ).trim()
          
          echo "All changed directories: ${changes}"
          
          // Split changes and filter valid services
          def changedList = changes.split("\n")
          def affected = changedList.findAll { service -> 
            env.VALID_SERVICES.split(",").contains(service) 
          }
          
          if (affected.isEmpty()) {
            currentBuild.result = 'SUCCESS'
            echo "No Laravel microservice changes detected. Skipping build."
            return
          }

          env.AFFECTED_SERVICES = affected.join(",")
          echo "Affected services: ${env.AFFECTED_SERVICES}"
        }
      }
    }

    stage('Build, Test & Deploy') {
      when {
        expression { return env.AFFECTED_SERVICES?.trim() }
      }
      steps {
        script {
          def services = env.AFFECTED_SERVICES.split(",")
          for (service in services) {
            dir(service) {
              /*stage("→ ${service}: Test") {
                sh 'cp .env.ci .env || true'
                sh 'composer install --no-interaction --prefer-dist'
                sh './vendor/bin/phpunit --testdox || true'
              }*/

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

              /*stage("→ ${service}: Deploy to K8s") {
                withCredentials([file(
                  credentialsId: "${KUBECONFIG_CREDENTIALS_ID}",
                  variable: 'KUBECONFIG'
                )]) {
                  def tag = "${env.BUILD_NUMBER}"
                  def image = "your-docker-repo/${service}:${tag}"
                  sh "kubectl --kubeconfig=$KUBECONFIG set image deployment/${service} ${service}=${image}"
                }
              }*/
            }
          }
        }
      }
    }
  }

  post {
    success {
      echo "✅ Auto-deployment complete for: ${env.AFFECTED_SERVICES ?: 'Nothing changed'}"
    }
    failure {
      echo "❌ Build failed. Check logs for details."
    }
  }
}