# pipeline
Pipeline for Jenkins

pipeline {
    agent any

    parameters {
        string(name: 'TARGET_SERVER', defaultValue: '', description: 'Enter the server hostname or IP to deploy to')
    }

    environment {
        ANSIBLE_USER = 'richardp'
        INVENTORY = "${params.TARGET_SERVER},"
        SERVER_LIST = "${params.TARGET_SERVER}"
        SSH_CHECK_PLAYBOOK = 'ssh-check.yml'
        UPDATE_PLAYBOOK = 'update.yml'
        REPORT_DIR = 'reports'
        REPORT_FILE = 'report.html'
    }

    stages {

        stage('Validate Server Input') {
            steps {
                script {
                    if (!params.TARGET_SERVER?.trim()) {
                        error "No server specified. Please provide a TARGET_SERVER to proceed."
                    }
                    echo "Server provided: ${params.TARGET_SERVER}"
                }
            }
        }

        stage('Pre-check SSH Connectivity') {
            steps {
                script {
                    echo "🔍 Running SSH connectivity check using Ansible playbook"

                    def verbosity = env.ANSIBLE_DEBUG == "true" ? "-vvv" : ""
                    def limitHosts = env.SERVER_LIST ? "--limit '${env.SERVER_LIST}'" : ""

                    sh """
                        ansible-playbook \
                            -i ${env.INVENTORY} \
                            -u ${env.ANSIBLE_USER} \
                            ${limitHosts} \
                            ${verbosity} \
                            ${env.SSH_CHECK_PLAYBOOK}
                    """
                }
            }
            post {
                failure {
                    echo "❌ SSH connectivity check failed. Investigate host reachability, SSH keys, or inventory configuration."
                }
                success {
                    echo "✅ SSH connectivity verified for all target hosts."
                }
            }
        }

        stage('Run Updates') {
            steps {
                script {
                    echo "🚀 Running update playbook"

                    sh "mkdir -p ${env.REPORT_DIR}"

                    def buildDate = sh(script: "date +%Y-%m-%d", returnStdout: true).trim()
                    def verbosity = env.ANSIBLE_DEBUG == "true" ? "-vvv" : ""
                    def limitHosts = env.SERVER_LIST ? "--limit '${env.SERVER_LIST}'" : ""

                    sh """
                        ansible-playbook \
                            -i ${env.INVENTORY} \
                            -u ${env.ANSIBLE_USER} \
                            ${limitHosts} \
                            ${verbosity} \
                            ${env.UPDATE_PLAYBOOK} \
                            --extra-vars="@deployment_vars/shell_global.yml" \
                            --extra-vars="code_user=${env.ANSIBLE_USER} \
                                          build_number=${env.BUILD_NUMBER} \
                                          build_date=${buildDate} \
                                          build_status=Success \
                                          target_hosts=${params.TARGET_SERVER}"
                    """
                }
            }
            post {
                failure {
                    echo "❌ Update playbook failed. Review logs and generated reports."
                }
                success {
                    echo "✅ Update playbook completed successfully."
                }
            }
        }

        stage('Publish Report') {
            steps {
                script {
                    echo "📄 Publishing update report from ${env.REPORT_DIR}/${env.REPORT_FILE}"

                    publishHTML(target: [
                        reportName : 'Update Report',
                        reportDir  : "${env.REPORT_DIR}",
                        reportFiles: "${env.REPORT_FILE}",
                        keepAll    : true,
                        allowMissing: false,
                        alwaysLinkToLastBuild: true
                    ])
                }
            }
        }
    }
}
