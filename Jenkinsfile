pipeline {
    agent any

    environment {
        TARGET_URL = 'http://nginx-proxy:80'
        ZAP_REPORT_DIR = '/home/dharm/webapp-security-project/zap-reports'
        COMPOSE_PROJECT_NAME = 'webapp-security-project'
    }

    stages {

        stage('Build') {
            steps {
                echo 'Pulling latest images...'
                sh 'cd /home/dharm/webapp-security-project && docker-compose pull juice-shop nginx'
            }
        }

        stage('Deploy') {
            steps {
                echo 'Starting Juice Shop + nginx...'
                sh 'cd /home/dharm/webapp-security-project && docker-compose up -d juice-shop nginx'
                sh 'sleep 10'
            }
        }

        stage('Security Scan') {
            steps {
                echo 'Running ZAP baseline scan...'
                sh """
                    mkdir -p ${ZAP_REPORT_DIR}
                    chmod 777 ${ZAP_REPORT_DIR}
                    docker run --rm --user root \
                      --network webapp-security-project_default \
                      -v ${ZAP_REPORT_DIR}:/zap/wrk/:rw \
                      ghcr.io/zaproxy/zaproxy:stable \
                      zap-baseline.py -t ${TARGET_URL} \
                      -r zap-report-\$(date +%Y%m%d-%H%M%S).html \
                      -I
                """
            }
        }

        stage('Ansible Remediate') {
            steps {
                echo 'Applying remediation playbook...'
                sh 'ansible-playbook /home/dharm/webapp-security-project/ansible/milestone9_remediation.yml'
            }
        }

        stage('Re-scan') {
            steps {
                echo 'Re-running ZAP scan to verify remediation...'
                sh """
                    docker run --rm --user root \
                      --network webapp-security-project_default \
                      -v ${ZAP_REPORT_DIR}:/zap/wrk/:rw \
                      ghcr.io/zaproxy/zaproxy:stable \
                      zap-baseline.py -t ${TARGET_URL} \
                      -r zap-rescan-\$(date +%Y%m%d-%H%M%S).html \
                      -I
                """
            }
        }

        stage('Report') {
            steps {
                echo 'Copying ZAP reports into workspace for archiving...'
                sh """
                    mkdir -p ${WORKSPACE}/zap-reports
                    cp ${ZAP_REPORT_DIR}/*.html ${WORKSPACE}/zap-reports/ || true
                """
                echo 'Archiving ZAP reports...'
                archiveArtifacts artifacts: 'zap-reports/*.html', fingerprint: true
            }
        }
    }

    post {
        always {
            echo 'Pipeline finished.'
        }
    }
}
