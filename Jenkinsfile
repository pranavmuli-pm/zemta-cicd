pipeline {
  agent any

  environment {
    BACKUP_DIR  = "/u1/backups"
    STAGING_DIR = "/u1/staging/ZEMTA"
    BACKUP_DATE = sh(script: "date +%Y-%m-%d", returnStdout: true).trim()
  }

  stages {

    stage('Read Manifest') {
      steps {
        sh '''
          echo "========== ZEMTA MANIFEST =========="
          jq . manifests/release_manifest_3.0.3.json
        '''
      }
    }

    stage('Validate Release Artifacts') {
      steps {
        sh '''
          RELEASE_HOST=$(jq -r '.artifacts.release_server.host' manifests/release_manifest_3.0.3.json)
          RELEASE_PATH=$(jq -r '.artifacts.release_server.path' manifests/release_manifest_3.0.3.json)

          FILES=$(jq -r '.artifacts.files | .mta_core, .mta_et, .admin_ui' manifests/release_manifest_3.0.3.json)

          ssh rocky@${RELEASE_HOST} bash -s -- "${RELEASE_PATH}" ${FILES} << 'EOF'
          set -e
          RELEASE_PATH="$1"
          shift

          for file in "$@"; do
            FULL="${RELEASE_PATH}/${file}"
            [ -f "$FULL" ] || { echo "[ERROR] Missing $FULL"; exit 1; }
            SIZE=$(stat -c%s "$FULL")
            [ "$SIZE" -gt 0 ] || { echo "[ERROR] Empty $FULL"; exit 1; }
            echo "[OK] $file size=$SIZE"
          done
EOF
        '''
      }
    }

    stage('Copy Artifacts to Target Nodes (DRY RUN)') {
      steps {
        sh '''
          VERSION=$(jq -r '.version' manifests/release_manifest_3.0.3.json)
          RELEASE_HOST=$(jq -r '.artifacts.release_server.host' manifests/release_manifest_3.0.3.json)
          RELEASE_PATH=$(jq -r '.artifacts.release_server.path' manifests/release_manifest_3.0.3.json)

          FILES=$(jq -r '.artifacts.files | .mta_core, .mta_et, .admin_ui' manifests/release_manifest_3.0.3.json)
          TARGETS=$(jq -r '.targets.mta_core[].host' manifests/release_manifest_3.0.3.json | sort -u)

          for host in $TARGETS; do
            echo ">>> Copying artifacts to $host (DRY RUN)"

            ssh rocky@$host "
              sudo mkdir -p ${STAGING_DIR}/${VERSION}
              sudo chown rocky:rocky ${STAGING_DIR}/${VERSION}
            "

            for file in $FILES; do
              echo "Copying $file to $host"
              scp rocky@${RELEASE_HOST}:${RELEASE_PATH}/${file} \
                  rocky@${host}:${STAGING_DIR}/${VERSION}/
            done

            ssh rocky@$host "
              echo '--- Files on $host ---'
              ls -lh ${STAGING_DIR}/${VERSION}
            "
          done
        '''
      }
    }
  }

  post {
    success {
      echo "✅ DRY RUN artifact copy completed successfully"
    }
    failure {
      echo "❌ Pipeline failed — no deployment actions taken"
    }
  }
}

