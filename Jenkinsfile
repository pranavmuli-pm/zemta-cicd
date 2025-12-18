pipeline {
  agent any

  environment {
    STAGING_DIR = "/u1/staging/ZEMTA"
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

          MTA_CORE=$(jq -r '.artifacts.files.mta_core' manifests/release_manifest_3.0.3.json)
          MTA_ET=$(jq -r '.artifacts.files.mta_et' manifests/release_manifest_3.0.3.json)
          UI_ZIP=$(jq -r '.artifacts.files.admin_ui' manifests/release_manifest_3.0.3.json)

          ssh rocky@${RELEASE_HOST} bash -s -- \
            "${RELEASE_PATH}" "${MTA_CORE}" "${MTA_ET}" "${UI_ZIP}" << 'EOF'
          set -e
          RELEASE_PATH="$1"; shift

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

    stage('Copy MTA Artifacts (MTA Nodes Only)') {
      steps {
        sh '''
          VERSION=$(jq -r '.version' manifests/release_manifest_3.0.3.json)
          RELEASE_HOST=$(jq -r '.artifacts.release_server.host' manifests/release_manifest_3.0.3.json)
          RELEASE_PATH=$(jq -r '.artifacts.release_server.path' manifests/release_manifest_3.0.3.json)
          MTA_CORE=$(jq -r '.artifacts.files.mta_core' manifests/release_manifest_3.0.3.json)

          TARGETS=$(jq -r '.targets.mta_core[].host' manifests/release_manifest_3.0.3.json)

          for host in $TARGETS; do
            echo ">>> Copying MTA artifact to $host"

            ssh rocky@$host "
              sudo mkdir -p ${STAGING_DIR}/${VERSION}
              sudo chown rocky:rocky ${STAGING_DIR}/${VERSION}
            "

            scp rocky@${RELEASE_HOST}:${RELEASE_PATH}/${MTA_CORE} \
                rocky@${host}:${STAGING_DIR}/${VERSION}/

            ssh rocky@$host "ls -lh ${STAGING_DIR}/${VERSION}"
          done
        '''
      }
    }

    stage('Copy ET Artifacts (ET Nodes Only)') {
      steps {
        sh '''
          VERSION=$(jq -r '.version' manifests/release_manifest_3.0.3.json)
          RELEASE_HOST=$(jq -r '.artifacts.release_server.host' manifests/release_manifest_3.0.3.json)
          RELEASE_PATH=$(jq -r '.artifacts.release_server.path' manifests/release_manifest_3.0.3.json)
          MTA_ET=$(jq -r '.artifacts.files.mta_et' manifests/release_manifest_3.0.3.json)

          TARGETS=$(jq -r '.targets.mta_et[].host' manifests/release_manifest_3.0.3.json)

          for host in $TARGETS; do
            echo ">>> Copying ET artifact to $host"

            ssh rocky@$host "
              sudo mkdir -p ${STAGING_DIR}/${VERSION}
              sudo chown rocky:rocky ${STAGING_DIR}/${VERSION}
            "

            scp rocky@${RELEASE_HOST}:${RELEASE_PATH}/${MTA_ET} \
                rocky@${host}:${STAGING_DIR}/${VERSION}/

            ssh rocky@$host "ls -lh ${STAGING_DIR}/${VERSION}"
          done
        '''
      }
    }

    stage('Copy Admin UI Artifact (UI Node Only)') {
      steps {
        sh '''
          VERSION=$(jq -r '.version' manifests/release_manifest_3.0.3.json)
          RELEASE_HOST=$(jq -r '.artifacts.release_server.host' manifests/release_manifest_3.0.3.json)
          RELEASE_PATH=$(jq -r '.artifacts.release_server.path' manifests/release_manifest_3.0.3.json)
          UI_ZIP=$(jq -r '.artifacts.files.admin_ui' manifests/release_manifest_3.0.3.json)

          TARGETS=$(jq -r '.targets.admin_ui[].host' manifests/release_manifest_3.0.3.json)

          for host in $TARGETS; do
            echo ">>> Copying Admin UI artifact to $host"

            ssh rocky@$host "
              sudo mkdir -p ${STAGING_DIR}/${VERSION}
              sudo chown rocky:rocky ${STAGING_DIR}/${VERSION}
            "

            scp rocky@${RELEASE_HOST}:${RELEASE_PATH}/${UI_ZIP} \
                rocky@${host}:${STAGING_DIR}/${VERSION}/

            ssh rocky@$host "ls -lh ${STAGING_DIR}/${VERSION}"
          done
        '''
      }
    }
  }

  post {
    success {
      echo "✅ DRY RUN artifact copy completed per node type"
    }
    failure {
      echo "❌ Pipeline failed — no deployment actions executed"
    }
  }
}

