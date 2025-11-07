pipeline {
  agent any

  environment {
    JFROG_SERVER = "https://cbunifydev.jfrog.io"  // JFrog Cloud URL with SAST
    JFROG_CLI_PATH = "${env.WORKSPACE}/jf"
    SAST_PROJECT_DIR = "${env.WORKSPACE}/vulnado" 
    JFROG_SERVER_ID = "cbunifydev"
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
            echo ":key: Configuring JFrog CLoud CLI with credentials..."
            ./jf config add cbjfrog-server-jenkins \
              --url=${JFROG_SERVER} \
              --user=$JF_USER \
              --password=$JF_PASS \
              --interactive=false || ./jf config use cbjfrog-server-jenkins

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
            ../jf aud . --sast --format sarif > ../jfrog-sarif-sast-results.sarif || true
          '''
        }
      }
    }

    stage('Display SARIF Output') {
      steps {
        sh '''
          echo "📜 SAST SARIF Output Preview:"
          head -n 50 jfrog-sarif-sast-results.sarif || echo "No SARIF output found."
        '''
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'jfrog-sarif-sast-results.sarif', fingerprint: true
      echo "📄 SARIF file archived as 'jfrog-sarif-sast-results.sarif'"
    }
  }
}
