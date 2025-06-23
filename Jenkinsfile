pipeline {
  agent any

  environment {
    GIT_REPO_URL = 'https://github.com/ali-rostom1/adtripy'
    DOCKER_CREDENTIALS_ID = 'dockerhub-credentials'
    //KUBECONFIG_CREDENTIALS_ID = 'kubeconfig-creds'
    VALID_SERVICES = 'auth-service', 'booking-service', 'payment-service'
  }

  stages {
    stage('Checkout') {
      steps {
        git branch: 'main', url: "${env.GIT_REPO_URL}"
      }
    }

    stage('Detect Changed Services') {
      steps {
        script {
          def changes = sh(
            script: "git diff --name-only origin/main...HEAD | cut -d/ -f1 | sort -u",
            returnStdout: true
          ).trim().split("\n")

          def affected = changes.findAll { service -> VALID_SERVICES.contains(service) }

          if (affected.isEmpty()) {
            currentBuild.result = 'SUCCESS'
            echo "No Laravel microservice changes detected. Skipping build."
            return
          }

          env.AFFECTED_SERVICES = affected.join(",")
        }
      }
    }

    stage('Build, Test & Deploy') {
      when {
        expression { return env.AFFECTED_SERVICES }
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
