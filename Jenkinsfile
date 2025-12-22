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

    MTA_CORE=$(jq -r .artifacts.files.mta_core manifests/release_manifest_3.0.3.json)
    MTA_ET=$(jq -r .artifacts.files.mta_et manifests/release_manifest_3.0.3.json)
    UI_ZIP=$(jq -r .artifacts.files.admin_ui manifests/release_manifest_3.0.3.json)

    echo "Validating artifacts on ${RELEASE_HOST}:${RELEASE_PATH}"

    ssh rocky@${RELEASE_HOST} "
      set -e
      for file in ${MTA_CORE} ${MTA_ET} ${UI_ZIP}; do
        FULL_PATH=${RELEASE_PATH}/\\$file

        if [ ! -f \\$FULL_PATH ]; then
          echo '[ERROR] Missing artifact:' \\$FULL_PATH
          exit 1
        fi

        SIZE=\\$(stat -c%s \\$FULL_PATH)
        if [ \\$SIZE -le 0 ]; then
          echo '[ERROR] Empty artifact:' \\$FULL_PATH
          exit 1
        fi

        echo '[OK]' \\$file 'size=' \\$SIZE
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
          else
            echo '[WARN] No MTA jars on $host'
          fi
        "
      done

      for host in $(jq -r '.targets.mta_et[].host' manifests/release_manifest_3.0.3.json); do
        ssh rocky@$host "
          if ls /u1/zemta-inbound-email-server/*.jar >/dev/null 2>&1; then
            sudo tar -czf /u1/backups/MTA-ET${VERSION}_BKP_${DATE}.tar.gz /u1/zemta-inbound-email-server/*.jar
          else
            echo '[WARN] No ET jars on $host'
          fi
        "
      done

      for host in $(jq -r '.targets.admin_ui[].host' manifests/release_manifest_3.0.3.json); do
        ssh rocky@$host "
          if [ -d /var/www/html/mta-admin ]; then
            sudo tar -czf /u1/backups/MTA-UI${VERSION}_BKP_${DATE}.tar.gz /var/www/html/mta-admin
          else
            echo '[WARN] No UI directory on $host'
          fi
        "
      done
    '''
  }

  stage('Copy Artifacts (Per Node Type)') {
    sh '''
      set -e
      VERSION=$(jq -r .version manifests/release_manifest_3.0.3.json)
      RELEASE_HOST=$(jq -r .artifacts.release_server.host manifests/release_manifest_3.0.3.json)
      RELEASE_PATH=$(jq -r .artifacts.release_server.path manifests/release_manifest_3.0.3.json)

      MTA_CORE=$(jq -r .artifacts.files.mta_core manifests/release_manifest_3.0.3.json)
      MTA_ET=$(jq -r .artifacts.files.mta_et manifests/release_manifest_3.0.3.json)
      UI_ZIP=$(jq -r .artifacts.files.admin_ui manifests/release_manifest_3.0.3.json)

      for host in $(jq -r '.targets.mta_core[].host' manifests/release_manifest_3.0.3.json); do
        ssh rocky@$host "sudo mkdir -p /u1/staging/ZEMTA/${VERSION} && sudo chown rocky:rocky /u1/staging/ZEMTA/${VERSION}"
        scp rocky@${RELEASE_HOST}:${RELEASE_PATH}/${MTA_CORE} rocky@$host:/u1/staging/ZEMTA/${VERSION}/
      done

      for host in $(jq -r '.targets.mta_et[].host' manifests/release_manifest_3.0.3.json); do
        ssh rocky@$host "sudo mkdir -p /u1/staging/ZEMTA/${VERSION} && sudo chown rocky:rocky /u1/staging/ZEMTA/${VERSION}"
        scp rocky@${RELEASE_HOST}:${RELEASE_PATH}/${MTA_ET} rocky@$host:/u1/staging/ZEMTA/${VERSION}/
      done

      for host in $(jq -r '.targets.admin_ui[].host' manifests/release_manifest_3.0.3.json); do
        ssh rocky@$host "sudo mkdir -p /u1/staging/ZEMTA/${VERSION} && sudo chown rocky:rocky /u1/staging/ZEMTA/${VERSION}"
        scp rocky@${RELEASE_HOST}:${RELEASE_PATH}/${UI_ZIP} rocky@$host:/u1/staging/ZEMTA/${VERSION}/
      done
    '''
  }

  stage('Extract Artifacts') {
    sh '''
      set -e
      VERSION=$(jq -r .version manifests/release_manifest_3.0.3.json)

      for host in $(jq -r '.targets.mta_core[].host' manifests/release_manifest_3.0.3.json); do
        ssh rocky@$host "
          if [ -f /u1/staging/ZEMTA/${VERSION}/MTA-${VERSION}.tar.gz ]; then
            sudo tar -xzf /u1/staging/ZEMTA/${VERSION}/MTA-${VERSION}.tar.gz -C /u1/zemta
          fi
        "
      done

      for host in $(jq -r '.targets.mta_et[].host' manifests/release_manifest_3.0.3.json); do
        ssh rocky@$host "
          if [ -f /u1/staging/ZEMTA/${VERSION}/MTA-ET-${VERSION}.tar.gz ]; then
            sudo tar -xzf /u1/staging/ZEMTA/${VERSION}/MTA-ET-${VERSION}.tar.gz -C /u1/zemta-inbound-email-server
          fi
        "
      done

      for host in $(jq -r '.targets.admin_ui[].host' manifests/release_manifest_3.0.3.json); do
        ssh rocky@$host "
          if [ -f /u1/staging/ZEMTA/${VERSION}/mta-admin.zip ]; then
            sudo unzip -o /u1/staging/ZEMTA/${VERSION}/mta-admin.zip -d /var/www/html/
          fi
        "
      done
    '''
  }

  stage('Controlled Restart + Verify') {
    sh '''
      set -e

      for host in $(jq -r '.targets.mta_core[].host' manifests/release_manifest_3.0.3.json); do
        ssh rocky@$host "
          sudo /u1/zemta/mtaadminserver.sh restart
          sudo /u1/zemta/mtaserver.sh restart
          sudo /u1/zemta/mtaadminserver.sh status
          sudo /u1/zemta/mtaserver.sh status
        "
      done

      for host in $(jq -r '.targets.mta_et[].host' manifests/release_manifest_3.0.3.json); do
        ssh rocky@$host "
          sudo /u1/zemta-inbound-email-server/etserver.sh restart || true
        "
      done

      for host in $(jq -r '.targets.admin_ui[].host' manifests/release_manifest_3.0.3.json); do
        ssh rocky@$host "
          sudo systemctl restart httpd || true
        "
      done
    '''
  }

  stage('FINAL STATUS') {
    echo '✅ FULL ZEMTA PIPELINE EXECUTED SUCCESSFULLY'
  }
}

stage('Health Check') {
  sh '''
    set -e
    echo "=============================="
    echo " ZEMTA HEALTH CHECK"
    echo "=============================="

    for host in $(jq -r '.targets.mta_core[].host' manifests/release_manifest_3.0.3.json); do
      echo ">>> Checking MTA health on $host"

      ssh rocky@$host '
        /u1/zemta/mtaadminserver.sh status
        /u1/zemta/mtaserver.sh status
      '
    done

    for host in $(jq -r '.targets.mta_et[].host' manifests/release_manifest_3.0.3.json); do
      echo ">>> Checking ET health on $host"

      ssh rocky@$host '
        /u1/zemta-eventserver/mtaeventprocessor.sh status
      '
    done

    for host in $(jq -r '.targets.admin_ui[].host' manifests/release_manifest_3.0.3.json); do
      echo ">>> Checking UI health on $host"

      ssh rocky@$host '
        systemctl status httpd || true
      '
    done

    echo "[OK] Health check completed"
  '''
}
stage('Rollback Approval') {
  when {
    failed()
  }
  input {
    message "Health check failed. Do you want to ROLLBACK ZEMTA?"
    ok "Proceed with Rollback"
  }
}
stage('Rollback Execution') {
  when {
    failed()
  }
  sh '''
    set -e
    echo "=============================="
    echo " ZEMTA ROLLBACK STARTED"
    echo "=============================="

    for host in $(jq -r '.targets.mta_core[].host' manifests/release_manifest_3.0.3.json); do
      echo ">>> Rolling back MTA on $host"

      ssh rocky@$host '
        LATEST_BKP=$(ls -t /u1/backups/MTA*_BKP_*.tar.gz | head -1)

        if [ -z "$LATEST_BKP" ]; then
          echo "[ERROR] No MTA backup found"
          exit 1
        fi

        echo "Restoring from $LATEST_BKP"
        sudo tar -xzf "$LATEST_BKP" -C /
        sudo /u1/zemta/mtaadminserver.sh restart
        sudo /u1/zemta/mtaserver.sh restart
      '
    done

    for host in $(jq -r '.targets.admin_ui[].host' manifests/release_manifest_3.0.3.json); do
      echo ">>> Rolling back UI on $host"

      ssh rocky@$host '
        LATEST_UI_BKP=$(ls -t /u1/backups/MTA-UI*_BKP_*.tar.gz | head -1)

        if [ -n "$LATEST_UI_BKP" ]; then
          sudo tar -xzf "$LATEST_UI_BKP" -C /
          sudo systemctl restart httpd || true
        fi
      '
    done

    echo "[OK] Rollback completed"
  '''
}

