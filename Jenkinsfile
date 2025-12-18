stage('Extract MTA Artifact (MTA Nodes Only)') {
  steps {
    sh '''
      VERSION=$(jq -r .version manifests/release_manifest_3.0.3.json)
      MTA_TAR=$(jq -r .artifacts.files.mta_core manifests/release_manifest_3.0.3.json)

      for host in $(jq -r '.targets.mta_core[].host' manifests/release_manifest_3.0.3.json); do
        echo ">>> Extracting MTA on $host"
        ssh rocky@$host "
          if [ -f /u1/staging/ZEMTA/$VERSION/$MTA_TAR ]; then
            sudo tar -xzf /u1/staging/ZEMTA/$VERSION/$MTA_TAR -C /u1/zemta
            echo '[OK] MTA extracted on $host'
          else
            echo '[WARN] MTA artifact missing on $host'
          fi
        "
      done
    '''
  }
}

