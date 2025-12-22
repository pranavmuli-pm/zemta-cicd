node {

  def MANIFEST = "manifests/release_manifest_3.0.3.json"

  /* ===============================
     READ MANIFEST
     =============================== */
  stage('Read Manifest') {
    sh """
      echo '========== ZEMTA MANIFEST =========='
      jq . ${MANIFEST}
    """
  }

  /* ===============================
     VALIDATE RELEASE ARTIFACTS
     =============================== */
  stage('Validate Release Artifacts') {
    sh """
      set -e

      RELEASE_HOST=\$(jq -r .artifacts.release_server.host ${MANIFEST})
      RELEASE_PATH=\$(jq -r .artifacts.release_server.path ${MANIFEST})

      echo "Release Host : \$RELEASE_HOST"
      echo "Release Path : \$RELEASE_PATH"

      ssh rocky@\${RELEASE_HOST} "
        set -e
        for file in \$(jq -r '.artifacts.files | .mta_core, .mta_et, .admin_ui' ${MANIFEST}); do
          FULL_PATH=\${RELEASE_PATH}/\$file

          if [ ! -f \"\$FULL_PATH\" ]; then
            echo '[ERROR] Missing artifact:' \$FULL_PATH
            exit 1
          fi

          SIZE=\$(stat -c%s \"\$FULL_PATH\")
          if [ \$SIZE -le 0 ]; then
            echo '[ERROR] Empty artifact:' \$FULL_PATH
            exit 1
          fi

          echo '[OK]' \$file 'size=' \$SIZE
        done
      "
    """
  }

  /* ===============================
     BACKUP EXISTING FILES
     =============================== */
  stage('Backup Existing Files') {
    sh """
      set -e
      VERSION=\$(jq -r .version ${MANIFEST})
      DATE=\$(date +%Y-%m-%d)

      # --- MTA BACKUP ---
      for host in \$(jq -r '.targets.mta_core[].host' ${MANIFEST}); do
        ssh rocky@\$host "
          if ls /u1/zemta/*.jar >/dev/null 2>&1; then
            sudo tar -czf /u1/backups/MTA\${VERSION}_BKP_\${DATE}.tar.gz /u1/zemta/*.jar
          else
            echo '[WARN] No MTA jars on' \$host
          fi
        "
      done

      # --- ET BACKUP ---
      for host in \$(jq -r '.targets.mta_et[].host' ${MANIFEST}); do
        ssh rocky@\$host "
          if ls /u1/zemta-inbound-email-server/*.jar >/dev/null 2>&1; then
            sudo tar -czf /u1/backups/MTA-ET\${VERSION}_BKP_\${DATE}.tar.gz /u1/zemta-inbound-email-server/*.jar
          else
            echo '[WARN] No ET jars on' \$host
          fi
        "
      done

      # --- UI BACKUP ---
      for host in \$(jq -r '.targets.admin_ui[].host' ${MANIFEST}); do
        ssh rocky@\$host "
          if [ -d /var/www/html/mta-admin ]; then
            sudo tar -czf /u1/backups/MTA-UI\${VERSION}_BKP_\${DATE}.tar.gz /var/www/html/mta-admin
          else
            echo '[WARN] No UI directory on' \$host
          fi
        "
      done
    """
  }

  /* ===============================
     COPY ARTIFACTS (PER NODE TYPE)
     =============================== */
  stage('Copy Artifacts (Per Node Type)') {
    sh """
      set -e
      VERSION=\$(jq -r .version ${MANIFEST})
      RELEASE_HOST=\$(jq -r .artifacts.release_server.host ${MANIFEST})
      RELEASE_PATH=\$(jq -r .artifacts.release_server.path ${MANIFEST})

      MTA_CORE=\$(jq -r .artifacts.files.mta_core ${MANIFEST})
      MTA_ET=\$(jq -r .artifacts.files.mta_et ${MANIFEST})
      UI_ZIP=\$(jq -r .artifacts.files.admin_ui ${MANIFEST})

      for host in \$(jq -r '.targets.mta_core[].host' ${MANIFEST}); do
        ssh rocky@\$host "sudo mkdir -p /u1/staging/ZEMTA/\$VERSION && sudo chown rocky:rocky /u1/staging/ZEMTA/\$VERSION"
        scp rocky@\${RELEASE_HOST}:\${RELEASE_PATH}/\${MTA_CORE} rocky@\$host:/u1/staging/ZEMTA/\$VERSION/
      done

      for host in \$(jq -r '.targets.mta_et[].host' ${MANIFEST}); do
        ssh rocky@\$host "sudo mkdir -p /u1/staging/ZEMTA/\$VERSION && sudo chown rocky:rocky /u1/staging/ZEMTA/\$VERSION"
        scp rocky@\${RELEASE_HOST}:\${RELEASE_PATH}/\${MTA_ET} rocky@\$host:/u1/staging/ZEMTA/\$VERSION/
      done

      for host in \$(jq -r '.targets.admin_ui[].host' ${MANIFEST}); do
        ssh rocky@\$host "sudo mkdir -p /u1/staging/ZEMTA/\$VERSION && sudo chown rocky:rocky /u1/staging/ZEMTA/\$VERSION"
        scp rocky@\${RELEASE_HOST}:\${RELEASE_PATH}/\${UI_ZIP} rocky@\$host:/u1/staging/ZEMTA/\$VERSION/
      done
    """
  }

  /* ===============================
     EXTRACT ARTIFACTS
     =============================== */
  stage('Extract Artifacts') {
    sh """
      set -e
      VERSION=\$(jq -r .version ${MANIFEST})

      for host in \$(jq -r '.targets.mta_core[].host' ${MANIFEST}); do
        ssh rocky@\$host "
          [ -f /u1/staging/ZEMTA/\$VERSION/*.tar.gz ] &&
          sudo tar -xzf /u1/staging/ZEMTA/\$VERSION/*.tar.gz -C /u1/zemta
        "
      done

      for host in \$(jq -r '.targets.admin_ui[].host' ${MANIFEST}); do
        ssh rocky@\$host "
          [ -f /u1/staging/ZEMTA/\$VERSION/*.zip ] &&
          sudo unzip -o /u1/staging/ZEMTA/\$VERSION/*.zip -d /var/www/html/
        "
      done
    """
  }

  /* ===============================
     CONTROLLED RESTART + VERIFY
     =============================== */
  stage('Controlled Restart + Verify') {
    sh """
      set -e

      for host in \$(jq -r '.targets.mta_core[].host' ${MANIFEST}); do
        ssh rocky@\$host "
          sudo /u1/zemta/mtaadminserver.sh restart
          sudo /u1/zemta/mtaserver.sh restart
          sudo /u1/zemta/mtaadminserver.sh status
          sudo /u1/zemta/mtaserver.sh status
        "
      done

      for host in \$(jq -r '.targets.admin_ui[].host' ${MANIFEST}); do
        ssh rocky@\$host "
          sudo systemctl restart httpd || true
        "
      done
    """
  }

  /* ===============================
     FINAL STATUS
     =============================== */
  stage('FINAL STATUS') {
    echo "✅ FULL ZEMTA PIPELINE EXECUTED SUCCESSFULLY"
  }
}

