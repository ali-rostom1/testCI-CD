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