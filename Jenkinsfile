pipeline {
    agent any
    
    environment {
        APP_NAME = "temp-angular"
        BUILD_VERSION = "${env.BUILD_NUMBER}"
        ARTIFACT_NAME = "${APP_NAME}-${BUILD_VERSION}.tar.gz"
        DEPLOY_PATH = "/var/www/html/${APP_NAME}"
    }
    
    stages {
        stage('Build') {
            steps {
                script {
                    // Install dependencies and build Angular app
                    sh 'npm install'
                    sh 'npm run build'
                    
                    // Create versioned artifact
                    sh "tar -czvf ${ARTIFACT_NAME} -C dist/${APP_NAME} ."
                    
                    // Archive the artifact
                    archiveArtifacts artifacts: "${ARTIFACT_NAME}", onlyIfSuccessful: true
                }
            }
        }
        
        stage('Test') {
            steps {
                sh 'npm test'
            }
        }
        
        stage('Deploy') {
            steps {
                script {
                    // Use Ansible to deploy
                    ansiblePlaybook(
                        playbook: 'deploy.yml',
                        inventory: 'inventory.ini',
                        extraVars: [
                            artifact_path: "${ARTIFACT_NAME}",
                            app_name: "${APP_NAME}",
                            build_version: "${BUILD_VERSION}",
                            deploy_path: "${DEPLOY_PATH}"
                        ]
                    )
                }
            }
        }
    }
    
    post {
        success {
            echo "Deployment of version ${BUILD_VERSION} completed successfully!"
        }
        failure {
            echo "Deployment failed for version ${BUILD_VERSION}"
        }
    }
}