pipeline {
  agent any

  environment {
    JFROG_SERVER = "https://cbjfrog.saas-preprod.beescloud.com"
    JFROG_CLI_PATH = "${env.WORKSPACE}/jf"
    SCA_PROJECT_DIR = "${env.WORKSPACE}/vulnado"
  }

  stages {
    stage('Install JFrog CLI') {
      steps {
        retry(3) {
          sh '''
            if [ ! -f "$WORKSPACE/jf" ]; then
              echo ":package: Downloading JFrog CLI..."
              curl -fL https://releases.jfrog.io/artifactory/jfrog-cli/v2-jf/latest/jfrog-cli-linux-amd64/jf -o jf
              chmod +x jf
            fi
            ./jf --version
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
            echo ":key: Configuring JFrog CLI with credentials..."
            ./jf config add cbjfrog-server-jenkins \
              --url=${JFROG_SERVER} \
              --user=$JF_USER \
              --password=$JF_PASS \
              --interactive=false || ./jf config use cbjfrog-server-jenkins
          '''
        }
      }
    }

    stage('Run SCA Scan on vulnado') {
      steps {
        dir("${env.SCA_PROJECT_DIR}") {
          sh '''
            echo ":mag: Running SCA scan on vulnado project..."
            ../jf audit . --sca --format sarif > ../jfrog-sarif-sca-results.sarif || true
          '''
        }
      }
    }

    stage('Display SARIF Output') {
      steps {
        sh 'cat jfrog-sarif-sca-results.sarif || echo "No SARIF output found."'
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'jfrog-sarif-sca-results.sarif', fingerprint: true
    }
  }
}