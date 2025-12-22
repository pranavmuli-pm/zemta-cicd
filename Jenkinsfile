node {

  def MANIFEST = "manifests/release_manifest_3.0.3.json"

  stage('Read Manifest') {
    sh '''
      echo "========== ZEMTA MANIFEST =========="
      jq . ${MANIFEST}
    '''
  }

  stage('Validate Release Artifacts') {
    sh '''
      set -e
      RELEASE_HOST=$(jq -r .artifacts.release_server.host ${MANIFEST})
      RELEASE_PATH=$(jq -r .artifacts.release_server.path ${MANIFEST})

      ssh rocky@$RELEASE_HOST "
        set -e
        for file in $(jq -r '.artifacts.files | .mta_core, .mta_et, .admin_ui' ${MANIFEST}); do
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
      VERSION=$(jq -r .version ${MANIFEST})
      DATE=$(date +%Y-%m-%d)

      for host in $(jq -r '.targets.mta_core[].host' ${MANIFEST}); do
        ssh rocky@$host "
          ls /u1/zemta/*.jar >/dev/null 2>&1 &&
          sudo tar -czf /u1/backups/MTA${VERSION}_BKP_${DATE}.tar.gz /u1/zemta/*.jar ||
          echo '[WARN] No MTA jars'
        "
      done

      for host in $(jq -r '.targets.mta_et[].host' ${MANIFEST}); do
        ssh rocky@$host "
          ls /u1/zemta-et/*.jar >/dev/null 2>&1 &&
          sudo tar -czf /u1/backups/ET${VERSION}_BKP_${DATE}.tar.gz /u1/zemta-et/*.jar ||
          echo '[WARN] No ET jars'
        "
      done

      for host in $(jq -r '.targets.admin_ui[].host' ${MANIFEST}); do
        ssh rocky@$host "
          [ -d /var/www/html/mta-admin ] &&
          sudo tar -czf /u1/backups/UI${VERSION}_BKP_${DATE}.tar.gz /var/www/html/mta-admin ||
          echo '[WARN] No UI dir'
        "
      done
    '''
  }

  stage('Copy Artifacts (Per Node Type)') {
    sh '''
      set -e
      VERSION=$(jq -r .version ${MANIFEST})
      RELEASE_HOST=$(jq -r .artifacts.release_server.host ${MANIFEST})
      RELEASE_PATH=$(jq -r .artifacts.release_server.path ${MANIFEST})

      for host in $(jq -r '.targets.mta_core[].host' ${MANIFEST}); do
        ssh rocky@$host "sudo mkdir -p /u1/staging/ZEMTA/$VERSION && sudo chown rocky:rocky /u1/staging/ZEMTA/$VERSION"
        scp rocky@$RELEASE_HOST:$RELEASE_PATH/$(jq -r .artifacts.files.mta_core ${MANIFEST}) rocky@$host:/u1/staging/ZEMTA/$VERSION/
      done

      for host in $(jq -r '.targets.mta_et[].host' ${MANIFEST}); do
        ssh rocky@$host "sudo mkdir -p /u1/staging/ZEMTA/$VERSION && sudo chown rocky:rocky /u1/staging/ZEMTA/$VERSION"
        scp rocky@$RELEASE_HOST:$RELEASE_PATH/$(jq -r .artifacts.files.mta_et ${MANIFEST}) rocky@$host:/u1/staging/ZEMTA/$VERSION/
      done

      for host in $(jq -r '.targets.admin_ui[].host' ${MANIFEST}); do
        ssh rocky@$host "sudo mkdir -p /u1/staging/ZEMTA/$VERSION && sudo chown rocky:rocky /u1/staging/ZEMTA/$VERSION"
        scp rocky@$RELEASE_HOST:$RELEASE_PATH/$(jq -r .artifacts.files.admin_ui ${MANIFEST}) rocky@$host:/u1/staging/ZEMTA/$VERSION/
      done
    '''
  }

  stage('Extract Artifacts') {
    sh '''
      set -e
      VERSION=$(jq -r .version ${MANIFEST})

      for host in $(jq -r '.targets.mta_core[].host' ${MANIFEST}); do
        ssh rocky@$host "sudo tar -xzf /u1/staging/ZEMTA/$VERSION/*.tar.gz -C /u1/zemta"
      done

      for host in $(jq -r '.targets.mta_et[].host' ${MANIFEST}); do
        ssh rocky@$host "sudo tar -xzf /u1/staging/ZEMTA/$VERSION/*.tar.gz -C /u1/zemta-et"
      done

      for host in $(jq -r '.targets.admin_ui[].host' ${MANIFEST}); do
        ssh rocky@$host "sudo unzip -o /u1/staging/ZEMTA/$VERSION/*.zip -d /var/www/html/"
      done
    '''
  }

  stage('Controlled Restart + Health Check') {
    sh '''
      set -e
      FAIL=0

      for host in $(jq -r '.targets.mta_core[].host' ${MANIFEST}); do
        ssh rocky@$host "
          sudo /u1/zemta/mtaadminserver.sh restart &&
          sudo /u1/zemta/mtaserver.sh restart &&
          sudo /u1/zemta/mtaadminserver.sh status &&
          sudo /u1/zemta/mtaserver.sh status
        " || FAIL=1
      done

      for host in $(jq -r '.targets.mta_et[].host' ${MANIFEST}); do
        ssh rocky@$host "/u1/zemta-et/etserver.sh status" || FAIL=1
      done

      for host in $(jq -r '.targets.admin_ui[].host' ${MANIFEST}); do
        ssh rocky@$host "curl -f http://localhost || exit 1" || FAIL=1
      done

      [ $FAIL -eq 0 ] || exit 99
    '''
  }

  stage('Manual Approval (Only If Failed)') {
    when {
      expression { currentBuild.currentResult == 'FAILURE' }
    }
    input message: '''
❌ HEALTH CHECK FAILED

Approve to perform ROLLBACK
or Abort to investigate manually.
'''
  }

  stage('Rollback (If Approved)') {
    sh '''
      set -e
      VERSION=$(jq -r .version ${MANIFEST})

      for host in $(jq -r '.targets.mta_core[].host' ${MANIFEST}); do
        ssh rocky@$host "
          sudo tar -xzf /u1/backups/MTA${VERSION}_BKP_*.tar.gz -C /u1/zemta &&
          sudo /u1/zemta/mtaadminserver.sh restart &&
          sudo /u1/zemta/mtaserver.sh restart
        "
      done

      for host in $(jq -r '.targets.admin_ui[].host' ${MANIFEST}); do
        ssh rocky@$host "
          sudo tar -xzf /u1/backups/UI${VERSION}_BKP_*.tar.gz -C /var/www/html &&
          sudo systemctl restart httpd || true
        "
      done
    '''
  }

  stage('FINAL STATUS') {
    echo '✅ ZEMTA FULL HARDENED PIPELINE COMPLETED'
  }
}

