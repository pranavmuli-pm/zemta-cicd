pipeline {
  agent any

  stages {

    stage('Checkout') {
      steps {
        echo "Code checked out from GitHub"
      }
    }

    stage('Read Manifest') {
      steps {
        sh '''
          echo "========== RELEASE MANIFEST =========="
          cat release_manifest.json
        '''
      }
    }

    stage('Dry Run - Target Loop') {
      steps {
        sh '''
          echo "========== STARTING DRY RUN =========="

          TARGETS=$(jq -r '.targets[].host' release_manifest.json)

          for host in $TARGETS; do
            echo "--------------------------------------"
            echo "Target: $host"
            ssh rocky@$host "
              echo 'Host:' \$(hostname)
              echo 'Mode: DRY RUN'
              echo 'No changes executed'
            "
          done
        '''
      }
    }
  }

  post {
    success {
      echo "✅ GitHub-based DRY RUN completed successfully"
    }
    failure {
      echo "❌ Pipeline failed"
    }
  }
}

