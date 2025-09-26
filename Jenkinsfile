pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
    // Keep builds for report history
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }

  environment {
    AUT_HOST = 'application'
    AUT_PORT = '3000'
    DOCKER_NETWORK = 'jenkins_net'
    OUT_DIR = 'out'
    REPORTS_DIR = 'reports'
    JMETER_IMAGE = 'jmeter-prom:latest'
    JMETER_PROM_PORT = '9270'
    JMETER_CONTAINER_NAME = 'jmeter-run'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        echo "Starting Performance Testing Pipeline for ${env.BRANCH_NAME}"
      }
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
          echo "Waiting for Application Under Test to be ready..."
          for i in {1..90}; do
            if curl -fsS http://${AUT_HOST}:${AUT_PORT}/health >/dev/null 2>&1; then
              echo 'AUT is ready'; exit 0
            fi
            echo "Attempt \$i/90: AUT not ready, waiting..."
            sleep 2
          done
          echo 'AUT not healthy after 3 minutes'; exit 1
        """
      }
    }

    stage('Run Performance Tests') {
      steps {
        sh """
          echo "=== Starting JMeter Performance Tests ==="
          # Clean and recreate output directory
          rm -rf ${OUT_DIR}
          mkdir -p ${OUT_DIR}

          # Clean previous container if any
          docker rm -f ${JMETER_CONTAINER_NAME} >/dev/null 2>&1 || true

          # Run JMeter tests with HTML report generation
          docker run --rm \
            --name ${JMETER_CONTAINER_NAME} \
            --network=${DOCKER_NETWORK} \
            -v "\$PWD/jmeter:/work/jmeter:ro" \
            -v "\$PWD/${OUT_DIR}:/work/out" \
            ${JMETER_IMAGE} -n \
              -t /work/jmeter/test-plan-enhanced.jmx \
              -l /work/out/results.jtl \
              -e -o /work/out/jmeter-report \
              -Jjmeter.save.saveservice.output_format=xml \
              -Jjmeter.save.saveservice.response_data=true \
              -Jjmeter.save.saveservice.samplerData=true \
              -Jjmeter.save.saveservice.responseHeaders=true

          # Verify results were generated
          if [ -f "${OUT_DIR}/results.jtl" ]; then
            echo "=== JMeter Test Results Generated Successfully ==="
            echo "Total lines in results: \$(wc -l < ${OUT_DIR}/results.jtl)"
            echo "Sample results:"
            head -5 ${OUT_DIR}/results.jtl
          else
            echo "ERROR: No results.jtl file generated"
            exit 1
          fi
        """
      }
    }

    stage('Generate Performance Reports') {
      steps {
        sh """
          echo "=== Generating Comprehensive Performance Reports ==="

          # Create reports directory
          mkdir -p ${REPORTS_DIR}/generated

          # Generate performance summary
          cat > ${REPORTS_DIR}/generated/performance_summary.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Performance Test Summary - Build #${BUILD_NUMBER}</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        .header { background-color: #f8f9fa; padding: 15px; border-radius: 5px; }
        .metrics { display: flex; flex-wrap: wrap; gap: 20px; margin: 20px 0; }
        .metric-card { background: #e3f2fd; padding: 15px; border-radius: 5px; min-width: 200px; }
        .success { color: #4caf50; } .warning { color: #ff9800; } .error { color: #f44336; }
        table { border-collapse: collapse; width: 100%; margin: 20px 0; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #f2f2f2; }
    </style>
</head>
<body>
    <div class="header">
        <h1>🚀 Performance Test Report</h1>
        <p><strong>Build:</strong> #${BUILD_NUMBER} | <strong>Branch:</strong> ${BRANCH_NAME} | <strong>Date:</strong> \$(date)</p>
        <p><strong>Test Environment:</strong> Docker Containerized | <strong>Application:</strong> E-commerce API</p>
    </div>
EOF

          # Calculate performance metrics from JTL file
          if [ -f "${OUT_DIR}/results.jtl" ]; then
            TOTAL_REQUESTS=\$(tail -n +2 "${OUT_DIR}/results.jtl" | wc -l)
            SUCCESS_REQUESTS=\$(tail -n +2 "${OUT_DIR}/results.jtl" | awk -F',' '\$8=="true"' | wc -l)
            ERROR_REQUESTS=\$(tail -n +2 "${OUT_DIR}/results.jtl" | awk -F',' '\$8=="false"' | wc -l)
            SUCCESS_RATE=\$(echo "scale=1; \$SUCCESS_REQUESTS * 100 / \$TOTAL_REQUESTS" | bc)
            ERROR_RATE=\$(echo "scale=1; \$ERROR_REQUESTS * 100 / \$TOTAL_REQUESTS" | bc)

            # Get response time statistics (in milliseconds, column 2)
            AVG_RESPONSE=\$(tail -n +2 "${OUT_DIR}/results.jtl" | awk -F',' '{sum+=\$2; count++} END {if(count>0) print int(sum/count); else print 0}')
            MIN_RESPONSE=\$(tail -n +2 "${OUT_DIR}/results.jtl" | awk -F',' 'NR==2{min=\$2} {if(\$2<min) min=\$2} END {print int(min)}')
            MAX_RESPONSE=\$(tail -n +2 "${OUT_DIR}/results.jtl" | awk -F',' '{if(\$2>max) max=\$2} END {print int(max)}')

            # Calculate test duration
            START_TIME=\$(tail -n +2 "${OUT_DIR}/results.jtl" | head -1 | awk -F',' '{print \$1}')
            END_TIME=\$(tail -n +2 "${OUT_DIR}/results.jtl" | tail -1 | awk -F',' '{print \$1}')
            DURATION=\$(echo "scale=1; (\$END_TIME - \$START_TIME) / 1000" | bc)
            THROUGHPUT=\$(echo "scale=2; \$TOTAL_REQUESTS / \$DURATION" | bc)

            cat >> ${REPORTS_DIR}/generated/performance_summary.html << EOF
    <div class="metrics">
        <div class="metric-card">
            <h3>📊 Test Volume</h3>
            <p><strong>\$TOTAL_REQUESTS</strong> Total Requests</p>
            <p><strong>\${DURATION}s</strong> Test Duration</p>
        </div>
        <div class="metric-card">
            <h3>✅ Success Rate</h3>
            <p class="success"><strong>\$SUCCESS_RATE%</strong> (\$SUCCESS_REQUESTS/\$TOTAL_REQUESTS)</p>
        </div>
        <div class="metric-card">
            <h3>⚡ Response Times</h3>
            <p><strong>\${AVG_RESPONSE}ms</strong> Average</p>
            <p><strong>\${MIN_RESPONSE}ms - \${MAX_RESPONSE}ms</strong> Range</p>
        </div>
        <div class="metric-card">
            <h3>🔥 Throughput</h3>
            <p><strong>\${THROUGHPUT} req/sec</strong></p>
        </div>
    </div>

    <h2>📈 Detailed Results</h2>
    <table>
        <tr><th>Metric</th><th>Value</th><th>Status</th></tr>
        <tr><td>Total Requests</td><td>\$TOTAL_REQUESTS</td><td class="success">✅</td></tr>
        <tr><td>Success Rate</td><td>\$SUCCESS_RATE%</td><td class="\$([ \${SUCCESS_RATE%.*} -ge 95 ] && echo success || echo warning)">📊</td></tr>
        <tr><td>Error Rate</td><td>\$ERROR_RATE%</td><td class="\$([ \${ERROR_RATE%.*} -le 5 ] && echo success || echo warning)">📊</td></tr>
        <tr><td>Average Response Time</td><td>\${AVG_RESPONSE}ms</td><td class="\$([ \$AVG_RESPONSE -le 500 ] && echo success || echo warning)">⚡</td></tr>
        <tr><td>Max Response Time</td><td>\${MAX_RESPONSE}ms</td><td class="\$([ \$MAX_RESPONSE -le 1000 ] && echo success || echo warning)">⚡</td></tr>
        <tr><td>Throughput</td><td>\${THROUGHPUT} req/sec</td><td class="success">🔥</td></tr>
    </table>

    <h2>🔗 Additional Reports</h2>
    <ul>
        <li><a href="jmeter-report/index.html">📊 Detailed JMeter HTML Report</a></li>
        <li><a href="results.jtl">📄 Raw Test Results (JTL)</a></li>
        <li><a href="../Technical_Report.md">📋 Technical Analysis Report</a></li>
        <li><a href="../Business_Report.md">💼 Business Impact Report</a></li>
    </ul>

    <h2>📊 Test Summary</h2>
    <p>This automated performance test executed a complete e-commerce user journey including authentication, product browsing, cart management, and checkout processes.</p>

    <div style="background-color: \$([ \${SUCCESS_RATE%.*} -ge 90 ] && echo '#d4edda' || echo '#f8d7da'); padding: 15px; border-radius: 5px; margin: 20px 0;">
        <strong>Overall Status:</strong> \$([ \${SUCCESS_RATE%.*} -ge 90 ] && echo '✅ PASS - System performing within acceptable parameters' || echo '⚠️ REVIEW NEEDED - Performance issues detected')
    </div>
EOF
          fi

          cat >> ${REPORTS_DIR}/generated/performance_summary.html << 'EOF'
</body>
</html>
EOF

          echo "Performance summary report generated"
        """
      }
    }

    stage('Collect Prometheus Metrics') {
      steps {
        script {
          try {
            sh """
              echo "=== Collecting Current Prometheus Metrics ==="
              # Wait a moment for metrics to be processed
              sleep 5

              # Collect current metrics from Prometheus
              curl -s "http://prometheus:9090/api/v1/query?query=http_request_duration_ms_count" | jq . > ${OUT_DIR}/prometheus_metrics.json || echo "Could not collect Prometheus metrics"

              # Generate metrics summary
              curl -s "http://prometheus:9090/api/v1/query?query=rate(http_request_duration_ms_count[5m])" | jq . > ${OUT_DIR}/prometheus_rates.json || echo "Could not collect rate metrics"

              echo "Prometheus metrics collected"
            """
          } catch (Exception e) {
            echo "Warning: Could not collect Prometheus metrics: ${e.message}"
          }
        }
      }
    }

    stage('Archive Results') {
      steps {
        script {
          echo "=== Archiving All Performance Testing Artifacts ==="

          // Archive JMeter results and reports
          if (fileExists("${OUT_DIR}/results.jtl")) {
            archiveArtifacts artifacts: "${OUT_DIR}/**", fingerprint: true
            echo "✅ JMeter results and HTML reports archived"
          }

          // Archive generated reports
          if (fileExists("${REPORTS_DIR}")) {
            archiveArtifacts artifacts: "${REPORTS_DIR}/**", fingerprint: true
            echo "✅ Performance analysis reports archived"
          }

          // Publish HTML reports
          publishHTML([
            allowMissing: false,
            alwaysLinkToLastBuild: true,
            keepAll: true,
            reportDir: "${OUT_DIR}/jmeter-report",
            reportFiles: 'index.html',
            reportName: 'JMeter Performance Report',
            reportTitles: 'JMeter HTML Dashboard'
          ])

          publishHTML([
            allowMissing: false,
            alwaysLinkToLastBuild: true,
            keepAll: true,
            reportDir: "${REPORTS_DIR}/generated",
            reportFiles: 'performance_summary.html',
            reportName: 'Performance Summary',
            reportTitles: 'Performance Test Summary'
          ])

          echo "✅ HTML reports published to Jenkins"
        }
      }
    }

    stage('Performance Analysis') {
      steps {
        script {
          if (fileExists("${OUT_DIR}/results.jtl")) {
            // Read and analyze results
            def results = sh(script: "tail -n +2 ${OUT_DIR}/results.jtl | wc -l", returnStdout: true).trim().toInteger()
            def errors = sh(script: "tail -n +2 ${OUT_DIR}/results.jtl | awk -F',' '\$8==\"false\"' | wc -l", returnStdout: true).trim().toInteger()
            def successRate = ((results - errors) * 100) / results

            echo "📊 Performance Test Results:"
            echo "   Total Requests: ${results}"
            echo "   Errors: ${errors}"
            echo "   Success Rate: ${successRate.round(1)}%"

            // Set build status based on results
            if (successRate >= 95) {
              currentBuild.result = 'SUCCESS'
              echo "✅ Performance test PASSED - Success rate: ${successRate.round(1)}%"
            } else if (successRate >= 80) {
              currentBuild.result = 'UNSTABLE'
              echo "⚠️  Performance test UNSTABLE - Success rate: ${successRate.round(1)}%"
            } else {
              currentBuild.result = 'FAILURE'
              echo "❌ Performance test FAILED - Success rate: ${successRate.round(1)}%"
            }

            // Add build description
            currentBuild.description = "Success: ${successRate.round(1)}% | Requests: ${results} | Errors: ${errors}"
          }
        }
      }
    }
  }

  post {
    always {
      echo "=== Performance Testing Pipeline Complete ==="
      echo "Build: ${env.BUILD_URL}"
      echo "Reports available in Jenkins artifacts and HTML publisher"
    }

    success {
      echo "✅ Performance testing completed successfully"
    }

    unstable {
      echo "⚠️  Performance testing completed with warnings"
    }

    failure {
      echo "❌ Performance testing failed"
      // Optional: Send notifications here
    }

    cleanup {
      // Clean up Docker containers
      sh """
        docker rm -f ${JMETER_CONTAINER_NAME} >/dev/null 2>&1 || true
        echo "Cleanup completed"
      """
    }
  }
}