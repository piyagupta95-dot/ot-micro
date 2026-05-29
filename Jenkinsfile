// Import your globally configured shared library
@Library('my-shared-library') _

import org.company.DeploymentManager

pipeline {
    agent any
    
    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: 'Select deployment environment target')
        string(name: 'IMAGE_TAG', defaultValue: 'build-' + env.BUILD_NUMBER, description: 'Docker Tag for the current run')
        string(name: 'STAGING_FALLBACK_TAG', defaultValue: 'v1.0.0-stable', description: 'Fallback tag for Staging environment rollback tracking')
    }

    stages {
        stage('Checkout Custom Source') {
            steps {
                // Pull code from your custom application GitHub repository
                checkout scm
            }
        }

        stage('Build Attendance Image') {
            steps {
                script {
                    echo "Building Docker image targeting the attendance microservice configuration..."
                    // Execute the docker build context natively from the attendance subfolder
                    // Replace 'attendance' with your service subdirectory if structured differently
                    sh "docker build -t attendance-api:${params.IMAGE_TAG} ./attendance"
                }
            }
        }

        stage('Execute Core Deployment Strategy') {
            steps {
                script {
                    // Instantiate the custom Groovy class via constructor
                    def manager = new DeploymentManager(this, params.ENVIRONMENT)
                    
                    try {
                        // 1. Run environment diagnostics
                        manager.validate()
                        
                        // 2. Trigger active container rotation
                        manager.deploy(params.IMAGE_TAG)
                        
                        // Tag current image as backup for Dev if the pipeline deployment succeeds
                        if (params.ENVIRONMENT == 'dev') {
                            sh "docker tag attendance-api:${params.IMAGE_TAG} attendance-api:dev-backup"
                        }
                    } 
                    catch (Exception err) {
                        echo "💥 Runtime deployment error trapped: ${err.getMessage()}"
                        
                        // 3. Trigger targeted safety rollback actions
                        manager.rollback(params.STAGING_FALLBACK_TAG)
                        
                        // Finalize status warning to Jenkins UI dashboard
                        currentBuild.result = 'FAILURE'
                        error("Deployment cycle terminated with failures. Rollback finalized.")
                    }
                }
            }
        }
    }
}
