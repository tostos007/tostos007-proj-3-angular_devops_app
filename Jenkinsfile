pipeline {
    agent any

    environment {
        APP_NAME = "temp-angular"
        BUILD_VERSION = "${env.BUILD_NUMBER}"
        ARTIFACT_NAME = "${APP_NAME}-${BUILD_VERSION}.tar.gz"
        DEPLOY_PATH = "/var/www/html/${APP_NAME}"
        ARTIFACT_FULL_PATH = "${env.WORKSPACE}/${ARTIFACT_NAME}"
    }

    parameters {
        string(name: 'VERSION', defaultValue: '1.0.1', description: 'Version to deploy')
        choice(name: 'DEPLOY_TYPE', choices: ['deploy', 'rollback'], description: 'Deploy new version or rollback to previous')
        string(name: 'ROLLBACK_VERSION', defaultValue: '', description: 'Version to rollback to (used only if DEPLOY_TYPE is rollback)')
    }

    stages {
        stage('Build') {
            when {
                expression { params.DEPLOY_TYPE == 'deploy' }
            }
            steps {
                script {
                    sh 'npm install --legacy-peer-deps'
                    sh 'npm run build'
                    sh "tar -czvf ${ARTIFACT_NAME} -C dist/${APP_NAME}/browser ."

                    // Save versioned copy
                    sh "mkdir -p artifacts/${params.VERSION}"
                    sh "cp ${ARTIFACT_NAME} artifacts/${params.VERSION}/${APP_NAME}-${params.VERSION}.tar.gz"

                    writeFile file: "artifacts/${params.VERSION}/version.txt", text: "${params.VERSION}"

                    archiveArtifacts artifacts: "${ARTIFACT_NAME}", onlyIfSuccessful: true
                    archiveArtifacts artifacts: "artifacts/${params.VERSION}/*", onlyIfSuccessful: true
                }
            }
        }

        stage('Deploy') {
            when {
                expression { params.DEPLOY_TYPE == 'deploy' }
            }
            steps {
                script {
                    ansiblePlaybook(
                        playbook: 'deploy.yml',
                        inventory: 'hosts.ini',
                        extraVars: [
                            provided_artifact_name: "${ARTIFACT_NAME}",
                            artifact_path: "${ARTIFACT_FULL_PATH}",
                            app_name: "${APP_NAME}",
                            build_version: "${BUILD_VERSION}",
                            deploy_path: "${DEPLOY_PATH}",
                            current_version: "${BUILD_VERSION}",
                            app_symlink: "${DEPLOY_PATH}/current"
                        ]
                    )
                }
            }
        }

        stage('Verify Rollback Version') {
            when {
                expression { params.DEPLOY_TYPE == 'rollback' }
            }
            steps {
                script {
                    def rollbackVer = params.ROLLBACK_VERSION?.trim()
                    if (!rollbackVer) {
                        error("ROLLBACK_VERSION must be provided for rollback!")
                    }

                    def artifactPath = "artifacts/${rollbackVer}/${APP_NAME}-${rollbackVer}.tar.gz"
                    echo "🔍 Checking for artifact at ${artifactPath}"

                    if (!fileExists(artifactPath)) {
                        error("❌ No archived artifact found at ${artifactPath}")
                    }

                    env.ROLLBACK_ARTIFACT_PATH = artifactPath
                    echo "✅ Found artifact for rollback: ${artifactPath}"
                }
            }
        }

        stage('Rollback') {
            when {
                expression { params.DEPLOY_TYPE == 'rollback' }
            }
            steps {
                script {
                    ansiblePlaybook(
                        playbook: 'rollback.yml',
                        inventory: 'inventory.ini',
                        extraVars: [
                            rollback_version: "${params.ROLLBACK_VERSION}",
                            app_name: "${APP_NAME}",
                            deploy_path: "${DEPLOY_PATH}",
                            artifact_path: "${env.ROLLBACK_ARTIFACT_PATH}"
                        ]
                    )
                }
            }
        }
    }

    post {
        success {
            script {
                if (params.DEPLOY_TYPE == 'rollback') {
                    echo "✅ Successfully rolled back to version ${params.ROLLBACK_VERSION}"
                } else {
                    echo "✅ Deployment of version ${BUILD_VERSION} completed successfully!"
                }
            }
        }
        failure {
            script {
                if (params.DEPLOY_TYPE == 'rollback') {
                    echo "❌ Rollback to version ${params.ROLLBACK_VERSION} failed"
                } else {
                    echo "❌ Deployment failed for version ${BUILD_VERSION}"
                }
            }
        }
    }
}
