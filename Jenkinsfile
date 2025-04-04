pipeline {
    agent any
    
    environment {
        APP_NAME = "angular-devops-app"
        BUILD_VERSION = "${BUILD_NUMBER}"
        DEPLOY_DIR = "/var/www/html/${APP_NAME}"
        ARTIFACT_NAME = "${APP_NAME}-${BUILD_VERSION}.tar.gz"
    }
    // what is wrong with trump
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
        
        stage('Deploy') {
            steps {
                script {
                    // Parameter for rollback (version to deploy)
                    def deployVersion = params.ROLLBACK_VERSION ?: BUILD_VERSION
                    
                    // Download the specified version artifact
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
                        playbook: 'deploy.yml',
                        inventory: 'inventory.ini',
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
    def build = currentBuild.rawBuild.project._getLastBuilds(10) // Get last 10 builds
    
    build.each { b ->
        if (b.result.toString() == 'SUCCESS') {
            versions << b.number.toString()
        }
    }
    
    return versions.join('\n')
}