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
          
          # Create a Docker volume for sharing files
          docker volume create jmeter-vol-\${BUILD_NUMBER} || true
          
          # Copy files to the Docker volume using a temporary container
          docker run --rm \
            -v jmeter-vol-\${BUILD_NUMBER}:/work/jmeter \
            -v "\$PWD/jmeter:/source:ro" \
            alpine:latest \
            cp -r /source/* /work/jmeter/
          
          # Verify files are in the volume
          echo "=== DEBUG: Volume contents ==="
          docker run --rm \
            -v jmeter-vol-\${BUILD_NUMBER}:/work/jmeter:ro \
            alpine:latest \
            ls -la /work/jmeter/
          
          # Clean previous ephemeral container if any
          docker rm -f ${JMETER_CONTAINER_NAME} >/dev/null 2>&1 || true

          # Run JMeter with the Docker volume
          docker run --rm \
            --name ${JMETER_CONTAINER_NAME} \
            --network=${DOCKER_NETWORK} \
            -p ${JMETER_PROM_PORT}:${JMETER_PROM_PORT} \
            -v jmeter-vol-\${BUILD_NUMBER}:/work/jmeter:ro \
            -v "\$PWD/${OUT_DIR}:/work/out" \
            ${JMETER_IMAGE} -n \
              -t /work/jmeter/test-plan.jmx \
              -Jhost=${AUT_HOST} -Jport=${AUT_PORT} \
              -l /work/out/results.jtl \
              -e -o /work/out/report
          
          # Cleanup Docker volume
          docker volume rm jmeter-vol-\${BUILD_NUMBER} || true
        """
      }
    }

    stage('Archive') {
      steps {
        archiveArtifacts artifacts: "${OUT_DIR}/**", fingerprint: true
      }
    }
  }

  post {
    always {
      echo "Build: ${env.BUILD_URL}"
    }
  }
}