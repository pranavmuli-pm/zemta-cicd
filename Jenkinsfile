node {

  def DEPLOY_FAILED = false

  try {

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

        ssh rocky@${RELEASE_HOST} "
          set -e
          for file in ${MTA_CORE} ${MTA_ET} ${UI_ZIP}; do
            FULL_PATH=${RELEASE_PATH}/\\$file
            [ -f \\$FULL_PATH ] || exit 1
            SIZE=\\$(stat -c%s \\$FULL_PATH)
            [ \\$SIZE -gt 0 ] || exit 1
            echo '[OK]' \\$file
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
            mkdir -p /u1/backups
            ls /u1/zemta/*.jar >/dev/null 2>&1 &&
            sudo tar -czf /u1/backups/MTA${VERSION}_BKP_${DATE}.tar.gz /u1/zemta/*.jar || true
          "
        done

        for host in $(jq -r '.targets.admin_ui[].host' manifests/release_manifest_3.0.3.json); do
          ssh rocky@$host "
            [ -d /var/www/html/mta-admin ] &&
            sudo tar -czf /u1/backups/MTA-UI${VERSION}_BKP_${DATE}.tar.gz /var/www/html/mta-admin || true
          "
        done
      '''
    }

    stage('Copy Artifacts') {
      sh '''
        set -e
        VERSION=$(jq -r .version manifests/release_manifest_3.0.3.json)
        RELEASE_HOST=$(jq -r .artifacts.release_server.host manifests/release_manifest_3.0.3.json)
        RELEASE_PATH=$(jq -r .artifacts.release_server.path manifests/release_manifest_3.0.3.json)
        MTA_CORE=$(jq -r .artifacts.files.mta_core manifests/release_manifest_3.0.3.json)
        UI_ZIP=$(jq -r .artifacts.files.admin_ui manifests/release_manifest_3.0.3.json)

        for host in $(jq -r '.targets.mta_core[].host' manifests/release_manifest_3.0.3.json); do
          ssh rocky@$host "mkdir -p /u1/staging/ZEMTA/${VERSION}"
          scp rocky@${RELEASE_HOST}:${RELEASE_PATH}/${MTA_CORE} rocky@$host:/u1/staging/ZEMTA/${VERSION}/
        done

        for host in $(jq -r '.targets.admin_ui[].host' manifests/release_manifest_3.0.3.json); do
          ssh rocky@$host "mkdir -p /u1/staging/ZEMTA/${VERSION}"
          scp rocky@${RELEASE_HOST}:${RELEASE_PATH}/${UI_ZIP} rocky@$host:/u1/staging/ZEMTA/${VERSION}/
        done
      '''
    }

    stage('Extract + Restart') {
      sh '''
        set -e
        VERSION=$(jq -r .version manifests/release_manifest_3.0.3.json)

        for host in $(jq -r '.targets.mta_core[].host' manifests/release_manifest_3.0.3.json); do
          ssh rocky@$host "
            sudo tar -xzf /u1/staging/ZEMTA/${VERSION}/MTA-${VERSION}.tar.gz -C /u1/zemta
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
            /u1/zemta/mtaadminserver.sh status
            /u1/zemta/mtaserver.sh status
          "
        done
      '''
    }

  } catch (err) {
    DEPLOY_FAILED = true
    currentBuild.result = 'FAILURE'
    echo "❌ DEPLOYMENT FAILED"
    throw err
  }

  if (DEPLOY_FAILED) {

    stage('Rollback Approval') {
      input message: "❌ Deployment failed. Proceed with ROLLBACK?", ok: "Rollback"
    }

    stage('Rollback Execution') {
      sh '''
        set -e
        for host in $(jq -r '.targets.mta_core[].host' manifests/release_manifest_3.0.3.json); do
          ssh rocky@$host "
            LATEST=$(ls -t /u1/backups/MTA*_BKP_*.tar.gz | head -1)
            sudo tar -xzf $LATEST -C /
            sudo /u1/zemta/mtaadminserver.sh restart
            sudo /u1/zemta/mtaserver.sh restart
          "
        done
      '''
    }
  }

  stage('FINAL STATUS') {
    echo DEPLOY_FAILED ? '❌ DEPLOYMENT FAILED (ROLLED BACK)' : '✅ DEPLOYMENT SUCCESSFUL'
  }
}

