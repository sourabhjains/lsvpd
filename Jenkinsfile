/*
 * Copyright (c) 2026  IBM Corporation.  All rights reserved.
 *
 * Jenkins Declarative Pipeline for lsvpd
 * Translates Travis CI workflow to Jenkins with matrix builds
 * Supports: Fedora 37, Ubuntu 24.04
 * Container Runtime: Docker/Podman (auto-detected)
 */

pipeline {
    agent {
        label 'ras_ci'
    }

    environment {
        // Git commit information
        GIT_COMMIT_MSG = ''
        
        // Container runtime (auto-detected)
        CONTAINER_CMD = ''
        
        // Credentials (loaded from Jenkins)
        GITHUB_TOKEN = credentials('github-token')
        FTP_ACCESS = credentials('ftp3access')
        GSA_USER = credentials('gsauser')
        GITHUB_USER = credentials('githubuser')
        IBM_CLOUD_KEY = credentials('ibmcloudkey')
        
        // Build configuration
        CCACHE_DIR = "${HOME}/.ccache"
        WORKSPACE_DIR = "${WORKSPACE}"
    }

    stages {
        stage('Init') {
            steps {
                script {
                    echo "=========================================="
                    echo "Initializing lsvpd CI/CD Pipeline"
                    echo "=========================================="
                    
                    // Get commit message
                    GIT_COMMIT_MSG = sh(
                        script: 'git log -1 --pretty=format:"%s" ${GIT_COMMIT}',
                        returnStdout: true
                    ).trim()
                    echo "Commit: ${GIT_COMMIT_MSG}"
                    
                    // Detect container runtime
                    CONTAINER_CMD = sh(
                        script: '''
                            if command -v docker &> /dev/null; then
                                echo "docker"
                            elif command -v podman &> /dev/null; then
                                echo "podman"
                            else
                                echo "none"
                            fi
                        ''',
                        returnStdout: true
                    ).trim()
                    
                    if (CONTAINER_CMD == "none") {
                        error "Neither Docker nor Podman is available on this node"
                    }
                    
                    echo "Container Runtime: ${CONTAINER_CMD}"
                    
                    // Install Python dependencies for potential post-processing
                    try {
                        sh "python3 -m pip install --user github-api-python git-python PyGithub || true"
                    } catch (Exception e) {
                        echo "Warning: Could not install Python dependencies: ${e.message}"
                    }
                }
            }
        }

        stage('Build Matrix') {
            matrix {
                axes {
                    axis {
                        name 'DISTRO'
                        values 'fedora-37', 'ubuntu-24.04'
                    }
                }
                
                stages {
                    stage('Build Image - ${DISTRO}') {
                        steps {
                            script {
                                echo "=========================================="
                                echo "Building ${DISTRO} container image"
                                echo "=========================================="
                                
                                sh """
                                    ${CONTAINER_CMD} build --pull \
                                        -t lsvpd-${DISTRO} \
                                        -f lsvpd-ci/Dockerfile-${DISTRO} \
                                        .
                                """
                            }
                        }
                    }

                    stage('Run Build - ${DISTRO}') {
                        steps {
                            script {
                                echo "=========================================="
                                echo "Running ${DISTRO} build"
                                echo "=========================================="
                                
                                // Create ccache directory if it doesn't exist
                                sh "mkdir -p ${CCACHE_DIR}"
                                
                                // Run the build container
                                sh """
                                    ${CONTAINER_CMD} run \
                                        --security-opt seccomp=unconfined \
                                        --volume ${CCACHE_DIR}:/root/.ccache \
                                        --volume ${WORKSPACE_DIR}:/build \
                                        --tmpfs /tmp/ \
                                        --rm -t lsvpd-${DISTRO} \
                                        bash -c "./lsvpd-ci/build-${DISTRO}.sh" 2>&1 | tee build-${DISTRO}.log
                                """
                            }
                        }
                    }

                    stage('Archive Artifacts - ${DISTRO}') {
                        steps {
                            script {
                                echo "=========================================="
                                echo "Archiving ${DISTRO} artifacts"
                                echo "=========================================="
                                
                                // Archive build logs
                                archiveArtifacts artifacts: "build-${DISTRO}.log", 
                                                allowEmptyArchive: true
                                
                                // Archive distribution-specific artifacts
                                if (DISTRO == 'fedora-37') {
                                    // Archive RPM packages
                                    sh """
                                        if [ -d ~/rpmbuild/RPMS ]; then
                                            mkdir -p artifacts/${DISTRO}
                                            find ~/rpmbuild/RPMS -name '*.rpm' -exec cp {} artifacts/${DISTRO}/ \\; || true
                                        fi
                                    """
                                    archiveArtifacts artifacts: "artifacts/${DISTRO}/*.rpm",
                                                    allowEmptyArchive: true
                                }
                                
                                // Archive tarballs (both distributions)
                                sh """
                                    mkdir -p artifacts/${DISTRO}
                                    find . -maxdepth 1 -name '*.tar.gz' -exec cp {} artifacts/${DISTRO}/ \\; || true
                                """
                                archiveArtifacts artifacts: "artifacts/${DISTRO}/*.tar.gz",
                                                allowEmptyArchive: true
                            }
                        }
                    }

                    stage('Cleanup - ${DISTRO}') {
                        steps {
                            script {
                                echo "Cleaning up ${DISTRO} container image"
                                sh "${CONTAINER_CMD} rmi lsvpd-${DISTRO} || true"
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo "=========================================="
            echo "Pipeline completed successfully!"
            echo "=========================================="
            echo "All distributions built successfully"
            echo "Artifacts archived for:"
            echo "  - Fedora 37"
            echo "  - Ubuntu 24.04"
            
            script {
                try {
                    def successMessage = """✅ *Build Successful* - lsvpd
*Job:* ${env.JOB_NAME}
*Build:* #${env.BUILD_NUMBER}
*Commit:* ${GIT_COMMIT_MSG}
*Distributions:* Fedora 37, Ubuntu 24.04
*Duration:* ${currentBuild.durationString.replace(' and counting', '')}
*Details:* ${env.BUILD_URL}"""
                    
                    slackSend(
                        color: 'good',
                        channel: 'C034F2650KC',
                        tokenCredentialId: 'btassslackToken',
                        teamDomain: 'ibm-systems-power',
                        message: successMessage
                    )
                    
                    // Upload build info file
                    slackUploadFile(
                        filePath: 'build-info.txt',
                        initialComment: 'Build information and logs',
                        channel: 'C034F2650KC',
                        credentialId: 'btassslackToken'
                    )
                } catch (Exception e) {
                    echo "Warning: Could not send Slack notification: ${e.message}"
                }
            }
        }
        
        failure {
            echo "=========================================="
            echo "Pipeline failed!"
            echo "=========================================="
            echo "Check build logs for details"
            echo "Commit: ${GIT_COMMIT_MSG}"
            
            script {
                try {
                    def failureMessage = """❌ *Build Failed* - lsvpd
*Job:* ${env.JOB_NAME}
*Build:* #${env.BUILD_NUMBER}
*Commit:* ${GIT_COMMIT_MSG}
*Failed Stage:* Check console output
*Duration:* ${currentBuild.durationString.replace(' and counting', '')}
*Details:* ${env.BUILD_URL}console"""
                    
                    slackSend(
                        color: 'danger',
                        channel: 'C034F2650KC',
                        tokenCredentialId: 'btassslackToken',
                        teamDomain: 'ibm-systems-power',
                        message: failureMessage
                    )
                    
                    // Upload build info file if it exists
                    if (fileExists('build-info.txt')) {
                        slackUploadFile(
                            filePath: 'build-info.txt',
                            initialComment: 'Build failure information',
                            channel: 'C034F2650KC',
                            credentialId: 'btassslackToken'
                        )
                    }
                } catch (Exception e) {
                    echo "Warning: Could not send Slack notification: ${e.message}"
                }
            }
        }
        
        unstable {
            echo "=========================================="
            echo "Pipeline unstable"
            echo "=========================================="
            echo "Some builds may have warnings"
            
            script {
                try {
                    def unstableMessage = """⚠️ *Build Unstable* - lsvpd
*Job:* ${env.JOB_NAME}
*Build:* #${env.BUILD_NUMBER}
*Commit:* ${GIT_COMMIT_MSG}
*Status:* Build completed with warnings
*Duration:* ${currentBuild.durationString.replace(' and counting', '')}
*Details:* ${env.BUILD_URL}"""
                    
                    slackSend(
                        color: 'warning',
                        channel: 'C034F2650KC',
                        tokenCredentialId: 'btassslackToken',
                        teamDomain: 'ibm-systems-power',
                        message: unstableMessage
                    )
                    
                    // Upload build info file
                    if (fileExists('build-info.txt')) {
                        slackUploadFile(
                            filePath: 'build-info.txt',
                            initialComment: 'Build warning information',
                            channel: 'C034F2650KC',
                            credentialId: 'btassslackToken'
                        )
                    }
                } catch (Exception e) {
                    echo "Warning: Could not send Slack notification: ${e.message}"
                }
            }
        }
        
        always {
            script {
                echo "=========================================="
                echo "Cleaning up workspace"
                echo "=========================================="
                
                // Clean up any remaining containers
                sh """
                    ${CONTAINER_CMD} ps -a | grep lsvpd- | awk '{print \$1}' | xargs -r ${CONTAINER_CMD} rm -f || true
                """
                
                // Clean up dangling images
                sh """
                    ${CONTAINER_CMD} images -f dangling=true -q | xargs -r ${CONTAINER_CMD} rmi || true
                """
                
                // Archive workspace info
                sh """
                    echo "Build Number: ${BUILD_NUMBER}" > build-info.txt
                    echo "Build URL: ${BUILD_URL}" >> build-info.txt
                    echo "Git Commit: ${GIT_COMMIT}" >> build-info.txt
                    echo "Git Commit Message: ${GIT_COMMIT_MSG}" >> build-info.txt
                    echo "Container Runtime: ${CONTAINER_CMD}" >> build-info.txt
                    echo "Build Date: \$(date)" >> build-info.txt
                """
                archiveArtifacts artifacts: 'build-info.txt', allowEmptyArchive: true
                
                // Clean workspace (optional - uncomment if needed)
                // cleanWs()
            }
        }
    }
}
