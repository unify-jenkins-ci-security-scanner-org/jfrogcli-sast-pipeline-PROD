pipeline {
  agent any

  nvironment {
    JFROG_SERVER = "https://cbunifydev.jfrog.io"  // JFrog Cloud URL with SAST
    JFROG_CLI_PATH = "${env.WORKSPACE}/jf"
    SAST_PROJECT_DIR = "${env.WORKSPACE}/vulnado" 
    JFROG_SERVER_ID = "cbjfrog-server-jenkins"   // Reusing the same config name
  }

  stages {
    stage('Install JFrog CLI') {
      steps {
        retry(3) {
          sh '''
            if [ ! -f "$WORKSPACE/jf" ]; then
              echo ":package: Downloading JFrog CLI from install-cli.jfrog.io..."
              curl -fL https://install-cli.jfrog.io | sh
              chmod +x jfrog
              mv jfrog jf  # Rename to 'jf' for consistency in pipeline
            fi
            ./jf --version
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
            ./jf c rm ${JFROG_SERVER_ID} --quiet || true

            # Add updated config for cbunifydev JFrog Cloud
            ./jf c add ${JFROG_SERVER_ID} \
              --url=${JFROG_SERVER} \
              --user=$JF_USER \
              --password=$JF_PASS \
              --interactive=false

            # Use this configuration
            ./jf c use ${JFROG_SERVER_ID}
            ./jf c show
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
          ./jf xr curl api/v1/feature_flags | grep -i sast || true

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
            ../jf aud --sast --format=sarif ./www-project-vulnerable-flask-app > flask_jfrog_sast.sarif || true
          '''
        }
      }
    }

    stage('Display SARIF Output') {
      steps {
        sh '''
          echo "📜 SAST SARIF Output Preview:"
          head -n 50 flask_jfrog_sast.sarif || echo "No SARIF output found."
        '''
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'flask_jfrog_sast.sarif', fingerprint: true
      echo "📄 SARIF file archived as 'flask_jfrog_sast.sarif'"
    }
  }
}
