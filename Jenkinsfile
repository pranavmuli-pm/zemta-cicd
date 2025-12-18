node {

    stage('Extract MTA Artifact (TEST ONLY)') {

        sh '''
          VERSION=$(jq -r .version manifests/release_manifest_3.0.3.json)
          MTA_CORE=$(jq -r .artifacts.files.mta_core manifests/release_manifest_3.0.3.json)

          echo "Version      : $VERSION"
          echo "MTA artifact : $MTA_CORE"

          for host in $(jq -r '.targets.mta_core[].host' manifests/release_manifest_3.0.3.json); do
            echo ">>> Extracting MTA on $host"

            ssh rocky@$host "
              if [ -f /u1/staging/ZEMTA/$VERSION/$MTA_CORE ]; then
                echo '[INFO] Artifact found, extracting...'
                sudo tar -xzf /u1/staging/ZEMTA/$VERSION/$MTA_CORE -C /u1/zemta
                echo '[OK] Extraction completed on $host'
              else
                echo '[ERROR] Artifact not found on $host'
                exit 1
              fi
            "
          done
        '''
    }

}

