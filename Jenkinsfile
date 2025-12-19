node {

  stage('Controlled Restart – MTA (DRY RUN)') {
    sh '''
      set -e

      VERSION=$(jq -r .version manifests/release_manifest_3.0.3.json)

      echo "=============================="
      echo " ZEMTA MTA RESTART (DRY RUN)"
      echo " Version : $VERSION"
      echo "=============================="

      for host in $(jq -r '.targets.mta_core[].host' manifests/release_manifest_3.0.3.json); do
        echo ""
        echo ">>> DRY RUN on MTA node: $host"
        echo "--------------------------------"
        echo "Would execute:"
        echo "  ssh rocky@$host sudo /u1/zemta/mtaadminserver.sh restart"
        echo "  ssh rocky@$host sudo /u1/zemta/mtaserver.sh restart"
        echo "--------------------------------"
      done

      echo ""
      echo "[DRY RUN COMPLETE] No services were restarted."
    '''
  }

}

