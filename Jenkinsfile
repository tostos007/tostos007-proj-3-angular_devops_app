pipeline {
  agent any
  parameters {
    string(name: 'ROLLBACK_VERSION', defaultValue: '', description: 'Enter version to rollback (leave empty for new deploy)')
  }
  stages {
    stage('Build') {
      steps {
        sh 'npm install'
        sh 'npm run build'
      }
    }
    stage('Package') {
      when {
        expression { return params.ROLLBACK_VERSION == '' }
      }
      steps {
        script {
          env.VERSION = "1.0.${env.BUILD_NUMBER}"
          sh 'tar -czvf angular-app-${VERSION}.tar.gz -C dist/angular-devops-app .'
          archiveArtifacts artifacts: "angular-app-${VERSION}.tar.gz"
        }
      }
    }
    stage('Deploy') {
      steps {
        script {
          def versionToDeploy = params.ROLLBACK_VERSION ?: env.VERSION
          sh "ansible-playbook -i inventory playbook.yml --extra-vars 'version=${versionToDeploy}'"
        }
      }
    }
  }
}
