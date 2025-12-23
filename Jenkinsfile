/* ===============================
   Jenkins Parameters (ONE TIME)
   =============================== */
properties([
  parameters([
    string(
      name: 'RELEASE_VERSION',
      defaultValue: '3.0.3',
      description: 'ZEMTA version to deploy (example: 3.0.3, 3.0.4)'
    )
  ])
])

node {

  def MANIFEST = "manifests/release_manifest_${params.RELEASE_VERSION}.json"
  def DEPLOY_FAILED = false

  try {

    stage('Manifest Check') {
      sh """
        echo "Using manifest: ${MANIFEST}"
        [ -f ${MANIFEST} ] || (echo '❌ Manifest not found' && exit 1)
      """
    }

    stage('Read Manifest') {
      sh """
        echo "========== ZEMTA MANIFEST =========="
        jq . ${MANIFEST}
      """
    }

    stage('Validate Release Artifacts') {
      sh """
        set -e
        RELEASE_HOST=\$(jq -r .artifacts.release_server.host ${MANIFEST})
        RELEASE_PATH=\$(jq -r .artifacts.release_server.path ${MANIFEST})
        MTA_CORE=\$(jq -r .artifacts.files.mta_core ${MANIFEST})
        MTA_ET=\$(jq -r .artifacts.files.mta_et ${MANIFEST})
        UI_ZIP=\$(jq -r .artifacts.files.admin_ui ${MANIFEST})

        ssh rocky@\${RELEASE_HOST} "
          set -e
          for file in \${MTA_CORE} \${MTA_ET} \${UI_ZIP}; do
            FULL_PATH=\${RELEASE_PATH}/\\\$file
            [ -f \\\$FULL_PATH ] || (echo 'Missing file:' \\\$FULL_PATH && exit 1)
            SIZE=\\\$(stat -c%s \\\$FULL_PATH)
            [ \\\$SIZE -gt 0 ] || (echo 'Empty file:' \\\$FULL_PATH && exit 1)
            echo '[OK]' \\\$file
          done
        "
      """
    }

    stage('Backup Existing Files') {
      sh """
        set -e
        VERSION=\$(jq -r .version ${MANIFEST})
        DATE=\$(date +%Y-%m-%d)

        for host in \$(jq -r '.targets.mta_core[].host' ${MANIFEST}); do
          ssh rocky@\${host} "
            mkdir -p /u1/backups
            ls /u1/zemta/*.jar >/dev/null 2>&1 &&
            sudo tar -czf /u1/backups/MTA\${VERSION}_BKP_\${DATE}.tar.gz /u1/zemta/*.jar || true
          "
        done
      """
    }

    stage('Copy Artifacts') {
      sh """
        set -e
        VERSION=\$(jq -r .version ${MANIFEST})
        RELEASE_HOST=\$(jq -r .artifacts.release_server.host ${MANIFEST})
        RELEASE_PATH=\$(jq -r .artifacts.release_server.path ${MANIFEST})
        MTA_CORE=\$(jq -r .artifacts.files.mta_core ${MANIFEST})
        UI_ZIP=\$(jq -r .artifacts.files.admin_ui ${MANIFEST})

        for host in \$(jq -r '.targets.mta_core[].host' ${MANIFEST}); do
          ssh rocky@\${host} "mkdir -p /u1/staging/ZEMTA/\${VERSION}"
          scp rocky@\${RELEASE_HOST}:\${RELEASE_PATH}/\${MTA_CORE} rocky@\${host}:/u1/staging/ZEMTA/\${VERSION}/
        done
      """
    }

    stage('Extract + Restart') {
      sh """
        set -e
        VERSION=\$(jq -r .version ${MANIFEST})

        for host in \$(jq -r '.targets.mta_core[].host' ${MANIFEST}); do
          ssh rocky@\${host} "
            sudo tar -xzf /u1/staging/ZEMTA/\${VERSION}/MTA-\${VERSION}.tar.gz -C /u1/zemta
            sudo /u1/zemta/mtaadminserver.sh restart
            sudo /u1/zemta/mtaserver.sh restart
          "
        done
      """
    }

    stage('Health Check') {
      sh """
        for host in \$(jq -r '.targets.mta_core[].host' ${MANIFEST}); do
          ssh rocky@\${host} "
            /u1/zemta/mtaadminserver.sh status
            /u1/zemta/mtaserver.sh status
          "
        done
      """
    }

  } catch (err) {
    DEPLOY_FAILED = true
    currentBuild.result = 'FAILURE'
    throw err
  }

  stage('FINAL STATUS') {
    echo DEPLOY_FAILED ? '❌ DEPLOYMENT FAILED' : '✅ DEPLOYMENT SUCCESSFUL'
  }
}

