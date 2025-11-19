pipeline {
  agent any

  environment {
    JFROG_SERVER = "https://cbunifydev.jfrog.io"  // JFrog Cloud URL with SAST
    JFROG_CLI_PATH = "${env.WORKSPACE}/jf"
    SAST_PROJECT_DIR = "${env.WORKSPACE}/WebGoat" // entire repo - "${env.WORKSPACE}" 
    JFROG_SERVER_ID = "cbjfrog-server-jenkins"   // Reusing the same config name
  }

  stages {
    stage('Install JFrog CLI') {
      steps {
        retry(3) {
          sh '''
            if [ ! -f "$JFROG_CLI_PATH" ]; then
              echo ":package: Downloading JFrog CLI from install-cli.jfrog.io..."
              curl -fL https://install-cli.jfrog.io | sh
              chmod +x jfrog
              mv jfrog "$JFROG_CLI_PATH"  # Rename to 'jf' for consistency in pipeline
            fi
            "$JFROG_CLI_PATH" --version
          '''
        }
      }
    }

    stage('Configure JFrog CLI') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'jfrog-cloud-credentials',
          usernameVariable: 'JF_USER',
          passwordVariable: 'JF_PASS'
        )]) {
          sh '''
            echo ":key: Ensuring JFrog CLI is configured for cbunifydev..."

            # Remove existing config (optional first-time cleanup)
            "$JFROG_CLI_PATH" c rm ${JFROG_SERVER_ID} --quiet || true

            # Add updated config for cbunifydev JFrog Cloud
            "$JFROG_CLI_PATH" c add ${JFROG_SERVER_ID} \
              --url=${JFROG_SERVER} \
              --user=$JF_USER \
              --password=$JF_PASS \
              --interactive=false

            # Use this configuration
            "$JFROG_CLI_PATH" c use ${JFROG_SERVER_ID}
            "$JFROG_CLI_PATH" rt ping
          '''
        }
      }
    }

    stage('Debug Project Directory') {
      steps {
        sh '''
          echo "🔍 Listing contents of project directory:"
          ls -la "${SAST_PROJECT_DIR}"
        '''
      }
    }

    stage('Debug Environment and Project') {
      steps {
        sh '''
          echo "🔍 Checking JFrog feature flags:"
          "$JFROG_CLI_PATH" xr curl api/v1/feature_flags | grep -i sast || true

          echo "🔍 Listing project files in ${SAST_PROJECT_DIR}:"
          ls -la "${SAST_PROJECT_DIR}"
        '''
      }
    }

    stage('Run SAST Scan') {
      steps {
        dir("${env.SAST_PROJECT_DIR}") {
          sh '''
            echo ":mag: Running JFrog SAST scan..."
            "$JFROG_CLI_PATH" aud --sast --format=sarif . > WebGoat_jfrog_sast.sarif || true
          '''
        }
      }
    }

    stage('Display SARIF Output') {
      steps {
        sh '''
          echo "📜 SAST SARIF Output Preview:"
          head -n 50 ${SAST_PROJECT_DIR}/WebGoat_jfrog_sast.sarif || echo "No SARIF output found."
        '''
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: '**/WebGoat_jfrog_sast.sarif', fingerprint: true
      echo "📄 SARIF file archived as 'WebGoat_jfrog_sast.sarif'"
    }
  }
}
 