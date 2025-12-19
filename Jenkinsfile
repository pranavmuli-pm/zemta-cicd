stage('Controlled Restart – MTA (REAL + VERIFY)') {
  sh '''
    set -e

    VERSION=$(jq -r .version manifests/release_manifest_3.0.3.json)

    echo "======================================"
    echo " ZEMTA MTA RESTART + STATUS CHECK"
    echo " Version : $VERSION"
    echo "======================================"

    for host in $(jq -r '.targets.mta_core[].host' manifests/release_manifest_3.0.3.json); do
      echo ""
      echo ">>> Restarting MTA services on $host"
      echo "--------------------------------------"

      ssh rocky@$host "sudo /u1/zemta/mtaadminserver.sh restart"
      ssh rocky@$host "sudo /u1/zemta/mtaserver.sh restart"

      echo ""
      echo ">>> Verifying MTA services on $host"
      echo "--------------------------------------"

      ssh rocky@$host "sudo /u1/zemta/mtaadminserver.sh status"
      ssh rocky@$host "sudo /u1/zemta/mtaserver.sh status"

      echo "--------------------------------------"
      echo "[OK] Restart + verification completed on $host"
    done
  '''
}

