pipeline {
  agent any

  options { timestamps(); ansiColor('xterm') }

  environment {
    AUT_HOST = 'application'
    AUT_PORT = '3000'
    DOCKER_NETWORK = 'jenkins_net'
    OUT_DIR = 'out'
    // Our custom JMeter image with Prometheus Listener
    JMETER_IMAGE = 'jmeter-prom:latest'
    JMETER_PROM_PORT = '9270'
    JMETER_CONTAINER_NAME = 'jmeter-run'
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build JMeter Image') {
      steps {
        sh """
          docker build -t ${JMETER_IMAGE} ./jmeter
        """
      }
    }

    stage('Wait for AUT') {
      steps {
        sh """
          for i in {1..90}; do
            if curl -fsS http://${AUT_HOST}:${AUT_PORT}/health >/dev/null 2>&1; then
              echo 'AUT is up'; exit 0
            fi
            echo 'Waiting AUT...'; sleep 2
          done
          echo 'AUT not healthy in time'; exit 1
        """
      }
    }

    // stage('Run JMeter (docker) with Prometheus listener') {
    //   steps {
    //     sh """
    //       mkdir -p ${OUT_DIR}
    //       # Clean previous ephemeral container if any
    //       docker rm -f ${JMETER_CONTAINER_NAME} >/dev/null 2>&1 || true

    //       docker run --rm \
    //         --name ${JMETER_CONTAINER_NAME} \
    //         --network=${DOCKER_NETWORK} \
    //         -p ${JMETER_PROM_PORT}:${JMETER_PROM_PORT} \
    //         -v "\$PWD/jmeter:/work/jmeter:ro" \
    //         -v "\$PWD/${OUT_DIR}:/work/out" \
    //         ${JMETER_IMAGE} -n \
    //           -t /work/jmeter/test-plan.jmx \
    //           -Jhost=${AUT_HOST} -Jport=${AUT_PORT} \
    //           -l /work/out/results.jtl \
    //           -e -o /work/out/report
    //     """
    //   }
    // }
    stage('Run JMeter (docker) with Prometheus listener') {
      steps {
        sh """
          # Clean and recreate output directory
          rm -rf ${OUT_DIR}
          mkdir -p ${OUT_DIR}
          
          # Clean previous ephemeral container if any
          docker rm -f ${JMETER_CONTAINER_NAME} >/dev/null 2>&1 || true

          # Create JMeter container without starting it
          docker create \
            --name ${JMETER_CONTAINER_NAME} \
            --network=${DOCKER_NETWORK} \
            -p ${JMETER_PROM_PORT}:${JMETER_PROM_PORT} \
            -v "\$PWD/${OUT_DIR}:/work/out" \
            ${JMETER_IMAGE} -n \
              -t /work/jmeter/test-plan.jmx \
              -Jhost=${AUT_HOST} -Jport=${AUT_PORT} \
              -l /work/out/results.jtl \
              -f \
              -e -o /work/out/report
          
          # Copy JMeter files into the container
          docker cp jmeter/. ${JMETER_CONTAINER_NAME}:/work/jmeter/
          
          # Verify files are copied
          echo "=== DEBUG: Container JMeter directory contents ==="
          docker exec ${JMETER_CONTAINER_NAME} ls -la /work/jmeter/ || echo "Could not list files"
          
          # Start the container and wait for it to complete
          set +e  # Don't fail immediately on error
          docker start -a ${JMETER_CONTAINER_NAME}
          JMETER_EXIT_CODE=\$?
          set -e  # Re-enable immediate failure
          
          echo "=== JMeter container exit code: \$JMETER_EXIT_CODE ==="
          
          # Check container logs if it failed
          if [ \$JMETER_EXIT_CODE -ne 0 ]; then
            echo "=== JMeter container logs ==="
            docker logs ${JMETER_CONTAINER_NAME} || true
          fi
          
          # Check what was generated
          echo "=== DEBUG: Output directory contents ==="
          ls -la ${OUT_DIR}/ || echo "Output directory is empty or doesn't exist"
          
          # Check if results file was created
          if [ -f "${OUT_DIR}/results.jtl" ]; then
            echo "=== JMeter results file found ==="
            head -10 ${OUT_DIR}/results.jtl || true
          else
            echo "=== No results.jtl file found ==="
          fi
          
          # Remove the container
          docker rm ${JMETER_CONTAINER_NAME} || true
          
          # Exit with JMeter's exit code
          exit \$JMETER_EXIT_CODE
        """
      }
    }

    stage('Archive') {
      steps {
        script {
          // Check if there are files to archive
          def outFiles = sh(script: "ls ${OUT_DIR}/ 2>/dev/null | wc -l", returnStdout: true).trim()
          if (outFiles.toInteger() > 0) {
            echo "Found ${outFiles} files to archive"
            archiveArtifacts artifacts: "${OUT_DIR}/**", fingerprint: true
          } else {
            echo "No output files found to archive"
          }
        }
      }
    }
  }

  post {
    always {
      echo "Build: ${env.BUILD_URL}"
    }
  }
}