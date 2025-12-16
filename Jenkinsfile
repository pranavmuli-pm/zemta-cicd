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
              mkdir -p ${BACKUP_BASE}/${TIMESTAMP}/mta_core
              tar -czvf ${BACKUP_BASE}/${TIMESTAMP}/mta_core/mta_core_backup.tar.gz \
                /u1/zemta/zemta-admin-*.jar \
                /u1/zemta/lib/james-server-protocols-smtp-*.jar
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
              mkdir -p ${BACKUP_BASE}/${TIMESTAMP}/mta_et
              tar -czvf ${BACKUP_BASE}/${TIMESTAMP}/mta_et/mta_et_backup.tar.gz \
                /u1/zemta-inbound-email-server/zemta-admin-*.jar \
                /u1/zemta-inbound-email-server/lib/james-server-protocols-smtp-*.jar \
                /u1/zemta-eventserver/lib/mta-event-processor-*.jar
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
              mkdir -p ${BACKUP_BASE}/${TIMESTAMP}/ui
              tar -czvf ${BACKUP_BASE}/${TIMESTAMP}/ui/ui_backup.tar.gz \
                /var/www/html/mta-admin
            "
          done
        '''
      }
    }
  }

  post {
    success {
      echo "✅ Backup completed successfully"
    }
    failure {
      echo "❌ Backup failed — deployment stopped"
    }
  }
}

