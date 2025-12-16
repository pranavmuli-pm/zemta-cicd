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
          jq . manifests/release_manifest_3.0.3.json
        '''
      }
    }

    stage('Execution Plan (DRY RUN)') {
      steps {
        sh '''
          echo "========== EXECUTION PLAN =========="

          echo ""
          echo "Release Server:"
          jq -r '.artifacts.release_server.host + " -> " + .artifacts.release_server.path' \
            manifests/release_manifest_3.0.3.json

          echo ""
          echo "------------------------------------"
          echo "MTA CORE NODES"
          jq -r '.targets.mta_core[].host' manifests/release_manifest_3.0.3.json | while read host; do
            echo " Node: $host"
            jq -r '.node_groups.mta_core_nodes.steps[]' \
              manifests/release_manifest_3.0.3.json | sed 's/^/   - /'
          done

          echo ""
          echo "------------------------------------"
          echo "MTA ET NODES"
          jq -r '.targets.mta_et[].host' manifests/release_manifest_3.0.3.json | while read host; do
            echo " Node: $host"
            jq -r '.node_groups.mta_et_nodes.steps[]' \
              manifests/release_manifest_3.0.3.json | sed 's/^/   - /'
          done

          echo ""
          echo "------------------------------------"
          echo "ADMIN UI NODE"
          jq -r '.targets.admin_ui[].host' manifests/release_manifest_3.0.3.json | while read host; do
            echo " Node: $host"
            jq -r '.node_groups.admin_ui_node.steps[]' \
              manifests/release_manifest_3.0.3.json | sed 's/^/   - /'
          done

          echo ""
          echo "========== DRY RUN COMPLETE =========="
          echo "No commands executed on servers"
        '''
      }
    }
  }

  post {
    success {
      echo "✅ Manifest execution plan generated successfully"
    }
    failure {
      echo "❌ Failed to generate execution plan"
    }
  }
}

