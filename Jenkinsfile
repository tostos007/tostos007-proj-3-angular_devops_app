pipeline {
    agent any
    
    environment {
        APP_NAME = "angular-devops-app"
        BUILD_VERSION = "${BUILD_NUMBER}"
        DEPLOY_DIR = "/var/www/html/${APP_NAME}"
        ARTIFACT_NAME = "${APP_NAME}-${BUILD_VERSION}.tar.gz"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh 'npm ci'  // Clean install for consistency
            }
        }
        
        stage('Build') {
            steps {
                sh 'npm run build'
                
                // Verify build output exists
                script {
                    def buildExists = fileExists "dist/${APP_NAME}/index.html"
                    if (!buildExists) {
                        error "Build failed - no output found in dist/${APP_NAME}"
                    }
                }
                
                // Create versioned artifact
                sh "tar -czvf ${ARTIFACT_NAME} -C dist/${APP_NAME} ."
                
                // Archive the artifact
                archiveArtifacts artifacts: "${ARTIFACT_NAME}", onlyIfSuccessful: true
            }
        }
        
        stage('Deploy') {
            steps {
                script {
                    // Parameter for rollback (version to deploy)
                    def deployVersion = params.ROLLBACK_VERSION ?: BUILD_VERSION
                    def artifact = "${APP_NAME}-${deployVersion}.tar.gz"
                    
                    // If not current build, get from archives
                    if (deployVersion != BUILD_VERSION) {
                        copyArtifacts(
                            projectName: env.JOB_NAME,
                            selector: specific("${deployVersion}"),
                            filter: artifact,
                            target: '.'
                        )
                    }
                    
                    // Execute Ansible playbook
                    ansiblePlaybook(
                        playbook: 'ansible/deploy.yml',
                        inventory: 'ansible/inventory.ini',
                        extraVars: [
                            app_name: APP_NAME,
                            artifact_name: artifact,
                            build_version: deployVersion
                        ]
                    )
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()  // Clean workspace after build
        }
    }
    
    parameters {
        choice(
            name: 'ROLLBACK_VERSION',
            choices: 'none\n' + listPreviousVersions(),
            description: 'Select a previous version to rollback to (or none for current build)'
        )
    }
}

// Helper function to list previous successful builds
def listPreviousVersions() {
    def versions = []
    def builds = currentBuild.rawBuild.getPreviousBuilds().take(10)  // Get last 10 builds
    
    builds.each { b ->
        if (b.getResult().toString() == 'SUCCESS') {
            versions << b.getNumber().toString()
        }
    }
    
    return versions.join('\n')
}