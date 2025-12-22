node {

  stage('Read Manifest') {
    sh '''
      echo "========== ZEMTA MANIFEST =========="
      jq . manifests/release_manifest_3.0.3.json
    '''
  }

  stage('Validate Release Artifacts') {
    sh '''
      set -e

      RELEASE_HOST=$(jq -r .artifacts.release_server.host manifests/release_manifest_3.0.3.json)
      RELEASE_PATH=$(jq -r .artifacts.release_server.path manifests/release_manifest_3.0.3.json)

      ssh rocky@$RELEASE_HOST "
        set -e
        for file in MTA-3.0.3.tar.gz MTA-ET-3.0.3.tar.gz mta-admin.zip; do
          FULL_PATH=$RELEASE_PATH/$file
          [ -f \$FULL_PATH ] || { echo '[ERROR] Missing:' \$FULL_PATH; exit 1; }
          SIZE=\$(stat -c%s \$FULL_PATH)
          [ \$SIZE -gt 0 ] || { echo '[ERROR] Empty:' \$FULL_PATH; exit 1; }
          echo '[OK]' \$file 'size=' \$SIZE
        done
      "
    '''
  }

  stage('Backup Existing Files') {
    sh '''
      set -e
      VERSION=$(jq -r .version manifests/release_manifest_3.0.3.json)
      DATE=$(date +%Y-%m-%d)

      for host in $(jq -r '.targets.mta_core[].host' manifests/release_manifest_3.0.3.json); do
        ssh rocky@$host "
          if ls /u1/zemta/*.jar >/dev/null 2>&1; then
            sudo tar -czf /u1/backups/MTA${VERSION}_BKP_${DATE}.tar.gz /u1/zemta/*.jar
          fi
        "
      done

      for host in $(jq -r '.targets.admin_ui[].host' manifests/release_manifest_3.0.3.json); do
        ssh rocky@$host "
          if [ -d /var/www/html ]; then
            sudo tar -czf /u1/backups/UI${VERSION}_BKP_${DATE}.tar.gz /var/www/html
          fi
        "
      done
    '''
  }

  stage('Copy Artifacts (Per Node Type)') {
    sh '''
      VERSION=$(jq -r .version manifests/release_manifest_3.0.3.json)
      RELEASE_HOST=$(jq -r .artifacts.release_server.host manifests/release_manifest_3.0.3.json)
      RELEASE_PATH=$(jq -r .artifacts.release_server.path manifests/release_manifest_3.0.3.json)

      for host in $(jq -r '.targets.mta_core[].host' manifests/release_manifest_3.0.3.json); do
        ssh rocky@$host "sudo mkdir -p /u1/staging/ZEMTA/$VERSION && sudo chown rocky:rocky /u1/staging/ZEMTA/$VERSION"
        scp rocky@$RELEASE_HOST:$RELEASE_PATH/MTA-3.0.3.tar.gz rocky@$host:/u1/staging/ZEMTA/$VERSION/
      done

      for host in $(jq -r '.targets.admin_ui[].host' manifests/release_manifest_3.0.3.json); do
        ssh rocky@$host "sudo mkdir -p /u1/staging/ZEMTA/$VERSION && sudo chown rocky:rocky /u1/staging/ZEMTA/$VERSION"
        scp rocky@$RELEASE_HOST:$RELEASE_PATH/mta-admin.zip rocky@$host:/u1/staging/ZEMTA/$VERSION/
      done
    '''
  }

  stage('Extract Artifacts') {
    sh '''
      VERSION=$(jq -r .version manifests/release_manifest_3.0.3.json)

      for host in $(jq -r '.targets.mta_core[].host' manifests/release_manifest_3.0.3.json); do
        ssh rocky@$host "
          sudo tar -xzf /u1/staging/ZEMTA/$VERSION/MTA-3.0.3.tar.gz -C /u1/zemta
        "
      done

      for host in $(jq -r '.targets.admin_ui[].host' manifests/release_manifest_3.0.3.json); do
        ssh rocky@$host "
          sudo unzip -o /u1/staging/ZEMTA/$VERSION/mta-admin.zip -d /var/www/html/
        "
      done
    '''
  }

  stage('Controlled Restart') {
    sh '''
      for host in $(jq -r '.targets.mta_core[].host' manifests/release_manifest_3.0.3.json); do
        ssh rocky@$host "
          sudo /u1/zemta/mtaadminserver.sh restart
          sudo /u1/zemta/mtaserver.sh restart
        "
      done
    '''
  }

  stage('Health Check') {
    sh '''
      set -e
      for host in $(jq -r '.targets.mta_core[].host' manifests/release_manifest_3.0.3.json); do
        ssh rocky@$host "
          sudo /u1/zemta/mtaadminserver.sh status
          sudo /u1/zemta/mtaserver.sh status
        "
      done
    '''
  }

  stage('Rollback (Manual Approval)') {
    input message: 'Health check failed? Do you want to ROLLBACK?', ok: 'Rollback Now'

    sh '''
      VERSION=$(jq -r .version manifests/release_manifest_3.0.3.json)

      for host in $(jq -r '.targets.mta_core[].host' manifests/release_manifest_3.0.3.json); do
        ssh rocky@$host "
          sudo tar -xzf /u1/backups/MTA${VERSION}_BKP_*.tar.gz -C /
          sudo /u1/zemta/mtaadminserver.sh restart
          sudo /u1/zemta/mtaserver.sh restart
        "
      done
    '''
  }

  stage('FINAL STATUS') {
    echo '✅ FULL ZEMTA PIPELINE COMPLETED (WITH HEALTH + ROLLBACK SAFETY)'
  }
}

