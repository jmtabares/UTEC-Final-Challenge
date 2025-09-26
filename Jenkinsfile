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
          mkdir -p ${OUT_DIR}
          
          # Debug: Check if files exist before mounting
          echo "=== DEBUG: Current directory ==="
          pwd
          echo "=== DEBUG: Workspace contents ==="
          ls -la
          echo "=== DEBUG: JMeter directory contents ==="
          ls -la jmeter/ || echo "jmeter directory not found"
          echo "=== DEBUG: test-plan.jmx file check ==="
          ls -la jmeter/test-plan.jmx || echo "test-plan.jmx not found"
          
          # Clean previous ephemeral container if any
          docker rm -f ${JMETER_CONTAINER_NAME} >/dev/null 2>&1 || true

          # Debug: Test the mount first
          echo "=== DEBUG: Testing Docker mount ==="
          docker run --rm \
            --network=${DOCKER_NETWORK} \
            -v "\$PWD/jmeter:/work/jmeter:ro" \
            alpine:latest \
            ls -la /work/jmeter/

          # Run JMeter
          docker run --rm \
            --name ${JMETER_CONTAINER_NAME} \
            --network=${DOCKER_NETWORK} \
            -p ${JMETER_PROM_PORT}:${JMETER_PROM_PORT} \
            -v "\$PWD/jmeter:/work/jmeter:ro" \
            -v "\$PWD/${OUT_DIR}:/work/out" \
            ${JMETER_IMAGE} -n \
              -t /work/jmeter/test-plan.jmx \
              -Jhost=${AUT_HOST} -Jport=${AUT_PORT} \
              -l /work/out/results.jtl \
              -e -o /work/out/report
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