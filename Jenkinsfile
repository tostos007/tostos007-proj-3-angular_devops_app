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
                    sh 'npm install --legacy-peer-deps'
                    sh 'npm run build'
                    sh "tar -czvf ${ARTIFACT_NAME} -C dist/${APP_NAME} ."
                    archiveArtifacts artifacts: "${ARTIFACT_NAME}", onlyIfSuccessful: true
                    
                    // Store version information for rollback
                    writeFile file: 'version.txt', text: "${BUILD_VERSION}"
                    archiveArtifacts artifacts: 'version.txt', onlyIfSuccessful: true
                }
            }
        }
        
        
        
        stage('Deploy') {
            steps {
                script {
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

// Rollback Pipeline
properties([
    parameters([
        choice(
            name: 'ROLLBACK_VERSION',
            choices: getBuildVersions(),
            description: 'Select version to rollback to'
        )
    ])
])

def getBuildVersions() {
    // In a real implementation, you would query your artifact storage
    // For this example, we'll simulate getting versions from build history
    
    def versions = []
    
    // Get successful builds and their version numbers
    def builds = Jenkins.instance.getItemByFullName(env.JOB_NAME).getBuilds()
    builds.each { build ->
        if (build.result == Result.SUCCESS) {
            def artifact = build.getArtifacts().find { it.relativePath == 'version.txt' }
            if (artifact) {
                def version = build.getArtifactManager().root().child(artifact.relativePath).readToString().trim()
                versions << version
            }
        }
    }
    
    return versions.size() > 0 ? versions.unique().join('\n') : '1' // Default if none found
}

pipeline {
    agent any
    
    parameters {
        choice(
            name: 'ROLLBACK_VERSION',
            choices: getBuildVersions(),
            description: 'Select version to rollback to'
        )
    }
    
    stages {
        stage('Verify Rollback Version') {
            steps {
                script {
                    echo "Preparing to rollback to version ${params.ROLLBACK_VERSION}"
                    
                    // Verify the artifact exists
                    def build = Jenkins.instance.getItemByFullName(env.JOB_NAME).getBuildByNumber(params.ROLLBACK_VERSION.toInteger())
                    if (!build) {
                        error("Build ${params.ROLLBACK_VERSION} not found")
                    }
                    
                    def artifact = build.getArtifacts().find { it.relativePath == "${APP_NAME}-${params.ROLLBACK_VERSION}.tar.gz" }
                    if (!artifact) {
                        error("Artifact for version ${params.ROLLBACK_VERSION} not found")
                    }
                }
            }
        }
        
        stage('Rollback') {
            steps {
                script {
                    ansiblePlaybook(
                        playbook: 'rollback.yml',
                        inventory: 'inventory.ini',
                        extraVars: [
                            rollback_version: "${params.ROLLBACK_VERSION}",
                            app_name: "${APP_NAME}",
                            deploy_path: "${DEPLOY_PATH}"
                        ]
                    )
                }
            }
        }
    }
    
    post {
        success {
            echo "Successfully rolled back to version ${params.ROLLBACK_VERSION}"
        }
        failure {
            echo "Rollback to version ${params.ROLLBACK_VERSION} failed"
        }
    }
}