pipeline {
  agent any

  environment {
    BACKUP_DIR = "/u1/backups"
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

      MTA_CORE=$(jq -r '.artifacts.files.mta_core' manifests/release_manifest_3.0.3.json)
      MTA_ET=$(jq -r '.artifacts.files.mta_et' manifests/release_manifest_3.0.3.json)
      UI_ZIP=$(jq -r '.artifacts.files.admin_ui' manifests/release_manifest_3.0.3.json)

      echo "Validating artifacts on $RELEASE_HOST:$RELEASE_PATH"

      ssh rocky@$RELEASE_HOST "
        set -e

        for file in $MTA_CORE $MTA_ET $UI_ZIP; do
          FULL_PATH=$RELEASE_PATH/\$file

          if [ ! -f \$FULL_PATH ]; then
            echo '[ERROR] Missing artifact:' \$FULL_PATH
            exit 1
          fi

          SIZE=\$(stat -c%s \$FULL_PATH)
          if [ \$SIZE -le 0 ]; then
            echo '[ERROR] Empty artifact:' \$FULL_PATH
            exit 1
          fi

          echo '[OK]' \$file 'exists, size=' \$SIZE
        done
      "
    '''
  }
}


    stage('Backup - MTA Nodes') {
      steps {
        sh '''
          VERSION=$(jq -r '.version' manifests/release_manifest_3.0.3.json)
          TARGETS=$(jq -r '.targets.mta_core[].host' manifests/release_manifest_3.0.3.json)

          for host in $TARGETS; do
            echo "Backing up MTA on $host"

            ssh rocky@$host "
              if ls /u1/zemta/*.jar 1>/dev/null 2>&1; then
                sudo tar -czvf ${BACKUP_DIR}/MTA${VERSION}_BKP_${BACKUP_DATE}.tar.gz \
                  /u1/zemta/*.jar
              else
                echo '[WARN] No MTA JARs found on '${host}', skipping'
              fi
            "
          done
        '''
      }
    }

    stage('Backup - ET Nodes') {
      steps {
        sh '''
          VERSION=$(jq -r '.version' manifests/release_manifest_3.0.3.json)
          TARGETS=$(jq -r '.targets.mta_et[].host' manifests/release_manifest_3.0.3.json)

          for host in $TARGETS; do
            echo "Backing up ET on $host"

            ssh rocky@$host "
              if ls /u1/zemta-inbound-email-server/*.jar 1>/dev/null 2>&1; then
                sudo tar -czvf ${BACKUP_DIR}/MTA-ET${VERSION}_BKP_${BACKUP_DATE}.tar.gz \
                  /u1/zemta-inbound-email-server/*.jar
              else
                echo '[WARN] No ET JARs found on '${host}', skipping'
              fi
            "
          done
        '''
      }
    }

    stage('Backup - Admin UI Node') {
      steps {
        sh '''
          VERSION=$(jq -r '.version' manifests/release_manifest_3.0.3.json)
          TARGETS=$(jq -r '.targets.admin_ui[].host' manifests/release_manifest_3.0.3.json)

          for host in $TARGETS; do
            echo "Backing up Admin UI on $host"

            ssh rocky@$host "
              if [ -d /var/www/html/mta-admin ]; then
                sudo tar -czvf ${BACKUP_DIR}/MTA-UI${VERSION}_BKP_${BACKUP_DATE}.tar.gz \
                  /var/www/html/mta-admin
              else
                echo '[WARN] Admin UI not found on '${host}', skipping'
              fi
            "
          done
        '''
      }
    }
  }

  post {
    success {
      echo "✅ ZEMTA backups completed successfully (production-style)"
    }
    failure {
      echo "❌ Backup failed — deployment stopped"
    }
  }
}

