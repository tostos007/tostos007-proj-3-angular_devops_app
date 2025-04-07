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
        choice(
            name: 'ROLLBACK_VERSION',
            choices: ['1'], // Can be updated dynamically with active choice script plugin or default '1'
            description: 'Select version to rollback to'
        )
    }

    stages {
        stage('Build') {
            when {
                expression { params.ROLLBACK_VERSION == '1' }
            }
            steps {
                script {
                    sh 'npm install --legacy-peer-deps'
                    sh 'npm run build'
                    sh "tar -czvf ${ARTIFACT_NAME} -C dist/${APP_NAME} ."
                    archiveArtifacts artifacts: "${ARTIFACT_NAME}", onlyIfSuccessful: true

                    writeFile file: 'version.txt', text: "${BUILD_VERSION}"
                    archiveArtifacts artifacts: 'version.txt', onlyIfSuccessful: true
                }
            }
        }

        stage('Deploy') {
            when {
                expression { params.ROLLBACK_VERSION == '1' }
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
                expression { params.ROLLBACK_VERSION != '1' }
            }
            steps {
                script {
                    echo "Preparing to rollback to version ${params.ROLLBACK_VERSION}"
                    def job = Jenkins.instance.getItemByFullName(env.JOB_NAME)
                    def build = job?.getBuildByNumber(params.ROLLBACK_VERSION.toInteger())
                    if (!build) {
                        error("Build ${params.ROLLBACK_VERSION} not found")
                    }
                    def artifact = build.getArtifacts().find {
                        it.relativePath == "${APP_NAME}-${params.ROLLBACK_VERSION}.tar.gz"
                    }
                    if (!artifact) {
                        error("Artifact for version ${params.ROLLBACK_VERSION} not found")
                    }
                }
            }
        }

        stage('Rollback') {
            when {
                expression { params.ROLLBACK_VERSION != '1' }
            }
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
            script {
                if (params.ROLLBACK_VERSION != '1') {
                    echo "Successfully rolled back to version ${params.ROLLBACK_VERSION}"
                } else {
                    echo "Deployment of version ${BUILD_VERSION} completed successfully!"
                }
            }
        }
        failure {
            script {
                if (params.ROLLBACK_VERSION != '1') {
                    echo "Rollback to version ${params.ROLLBACK_VERSION} failed"
                } else {
                    echo "Deployment failed for version ${BUILD_VERSION}"
                }
            }
        }
    }
}
