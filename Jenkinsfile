pipeline {
  agent any

  environment {
    JFROG_USERNAME = credentials('jfrog-cli-credentials')
    IMAGE_TAR = "${env.WORKSPACE}/image.tar"
    JFROG_SERVER = "https://cbjfrog.saas-preprod.beescloud.com"
    JFROG_CLI_PATH = "${env.WORKSPACE}/jf"
  }

  stages {
    stage('Install JFrog CLI') {
      steps {
        retry(3) {
          sh '''
            if [ ! -f "$WORKSPACE/jf" ]; then
              echo ":package: Installing JFrog CLI via official installer..."
              curl -fL https://install-cli.jfrog.io | sh
              mv jf "$WORKSPACE/jf"
              chmod +x "$WORKSPACE/jf"
            fi
            "$WORKSPACE/jf" --version
          '''
        }
      }
    }

    stage('Configure JFrog CLI') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'jfrog-cli-credentials',
          usernameVariable: 'JF_USER',
          passwordVariable: 'JF_PASS'
        )]) {
          sh '''
            echo ":key: Configuring JFrog CLI with provided credentials..."
            "$WORKSPACE/jf" config add cbjfrog-server \
              --url=${JFROG_SERVER} \
              --user=$JF_USER \
              --password=$JF_PASS \
              --interactive=false
          '''
        }
      }
    }

    stage('Scan Image with JFrog CLI') {
      steps {
        sh '''
          echo ":mag: Scanning image.tar using JFrog CLI..."
          "$WORKSPACE/jf" scan "${IMAGE_TAR}" --format=sarif > jfrog-sarif-results.sarif || true
        '''
      }
    }

    stage('Display SARIF Output') {
      steps {
        sh 'cat jfrog-sarif-results.sarif'
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'jfrog-sarif-results.sarif', fingerprint: true
    }
  }
}
