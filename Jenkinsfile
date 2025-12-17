pipeline {
  agent any

  environment {
    BACKUP_BASE = "/u1/backups/zemta"
    TIMESTAMP = sh(script: "date +%Y%m%d_%H%M%S", returnStdout: true).trim()
  }

  stages {

    stage('Checkout') {
      steps {
        echo "Code checked out from GitHub"
      }
    }

    stage('Read Manifest') {
      steps {
        sh '''
          echo "========== MANIFEST =========="
          jq . manifests/release_manifest_3.0.3.json
        '''
      }
    }

    stage('Backup - MTA Core Nodes') {
      steps {
        sh '''
          TARGETS=$(jq -r '.targets.mta_core[].host' manifests/release_manifest_3.0.3.json)

          for host in $TARGETS; do
            echo "Backing up MTA Core on $host"

            ssh rocky@$host "
              sudo mkdir -p ${BACKUP_BASE}/${TIMESTAMP}/mta_core

              if ls /u1/zemta/*.jar 1>/dev/null 2>&1; then
                sudo tar -czvf ${BACKUP_BASE}/${TIMESTAMP}/mta_core/mta_core_backup.tar.gz \
                  /u1/zemta/*.jar
              else
                echo '[WARN] No MTA Core JARs found, skipping backup'
              fi
            "
          done
        '''
      }
    }

    stage('Backup - MTA ET Nodes') {
      steps {
        sh '''
          TARGETS=$(jq -r '.targets.mta_et[].host' manifests/release_manifest_3.0.3.json)

          for host in $TARGETS; do
            echo "Backing up MTA ET on $host"

            ssh rocky@$host "
              sudo mkdir -p ${BACKUP_BASE}/${TIMESTAMP}/mta_et

              if ls /u1/zemta-inbound-email-server/*.jar 1>/dev/null 2>&1; then
                sudo tar -czvf ${BACKUP_BASE}/${TIMESTAMP}/mta_et/mta_et_backup.tar.gz \
                  /u1/zemta-inbound-email-server/*.jar
              else
                echo '[WARN] No MTA ET JARs found, skipping backup'
              fi
            "
          done
        '''
      }
    }

    stage('Backup - Admin UI') {
      steps {
        sh '''
          TARGETS=$(jq -r '.targets.admin_ui[].host' manifests/release_manifest_3.0.3.json)

          for host in $TARGETS; do
            echo "Backing up Admin UI on $host"

            ssh rocky@$host "
              sudo mkdir -p ${BACKUP_BASE}/${TIMESTAMP}/ui

              if [ -d /var/www/html/mta-admin ]; then
                sudo tar -czvf ${BACKUP_BASE}/${TIMESTAMP}/ui/ui_backup.tar.gz \
                  /var/www/html/mta-admin
              else
                echo '[WARN] Admin UI directory not found, skipping backup'
              fi
            "
          done
        '''
      }
    }
  }

  post {
    success {
      echo "✅ Backup completed successfully (tolerant mode)"
    }
    failure {
      echo "❌ Backup failed — pipeline stopped"
    }
  }
}

