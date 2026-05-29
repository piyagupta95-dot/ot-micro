// Import your custom shared library from Jenkins global system settings
@Library('company-deployment-lib') _

import org.company.DeploymentManager

pipeline {
    agent any
    
    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: 'Target lifecycle stage tier for deployment execution')
        string(name: 'IMAGE_TAG', defaultValue: 'build-' + env.BUILD_NUMBER, description: 'Compilation tag version metadata for attendance image')
        string(name: 'STAGING_FALLBACK_TAG', defaultValue: 'v1.0.0-stable', description: 'Explicit target deployment version to roll back to if Staging execution drops')
    }

    stages {
        stage('Sourcing SCM Workspace') {
            steps {
                // Pull source from your custom application repo
                checkout scm
            }
        }

        stage('Build Attendance Image') {
            steps {
                script {
                    echo "🛠️ Initiating container image assembly targeting the 'attendance' context folder..."
                    // Execute build context leveraging the attendance microservice configuration files
                    sh "docker build -t attendance-api:${params.IMAGE_TAG} ./attendance"
                }
            }
        }

        stage('Execute Core Deployment Lifecycle') {
            steps {
                script {
                    // Instantiate object using class constructor matching runtime pipeline context parameter
                    def manager = new DeploymentManager(this, params.ENVIRONMENT)
                    
                    try {
                        // 1. Run environment diagnostic validation
                        manager.validate()
                        
                        // 2. Perform fresh environment rolling upgrade deploy 
                        manager.deploy(params.IMAGE_TAG)
                        
                        // If DEV deploy finishes cleanly, preserve it to serve as the future instant local fallback cache point
                        if (params.ENVIRONMENT == 'dev') {
                            sh "docker tag attendance-api:${params.IMAGE_TAG} attendance-api:dev-backup"
                        }
                    } 
                    catch (Exception deploymentError) {
                        echo "💥 Deployment run anomaly trapped: ${deploymentError.getMessage()}"
                        
                        // 3. Initiate context rollback actions handling target stage environment strategies
                        manager.rollback(params.STAGING_FALLBACK_TAG)
                        
                        // Explicitly fail the job status so Jenkins reports a visual warning interface notification
                        currentBuild.result = 'FAILURE'
                        error("Lifecycle execution interrupted. Environment rolled back successfully to stable states.")
                    }
                }
            }
        }
    }
}
