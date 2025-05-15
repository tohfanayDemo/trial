pipeline {
    agent any

    parameters {
        choice(name: 'TEST_ENV', choices: ['QA', 'DEV', 'STAGE'], description: 'Choose the test environment')
    }

    environment {
        EMAIL_SUBJECT = "🧪 Booking API Test Report - ${env.JOB_NAME} #${env.BUILD_TAG}"
        RECIPIENTS = "team@example.com"
		 THRESHOLD = 80
    }
	
	options {
        skipDefaultCheckout(true)
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/your-org/postman-api-tests.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Set Credentials for Selected Environment') {
            steps {
                script {
                    // Dynamically assign credentials based on test environment
                    if (params.TEST_ENV == 'QA') {
                        env.CREDS_ID = 'qa-creds'
                    } else if (params.TEST_ENV == 'DEV') {
                        env.CREDS_ID = 'dev-creds'
                    } else if (params.TEST_ENV == 'STAGE') {
                        env.CREDS_ID = 'stage-creds'
                    }
                }
            }
        }

        stage('Run Newman Tests') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: "${env.CREDS_ID}", usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                        sh """
                            newman run "Booking Duplicate.postman_collection.json" \
                            -e "Booking_dynamic1.postman_environment.json" \
                            -d "small_subset.json" \
                            -r cli,json,junit,html,htmlextra \
                            --reporter-json-export reports/report.json \
                            --reporter-htmlextra-export reports/htmlextra_report.html \
                            --disable-unicode \
                            --env-var test_env=${params.TEST_ENV} \
                            --env-var cmd_username=$USERNAME \
                            --env-var cmd_password=$PASSWORD
                        """
                    }
                }
            }
        }
		
		stage('Evaluate Pass Threshold') {
            steps {
                script {
                    def report = readJSON file: 'reports/report.json'
                    def total = report.run.stats.tests.total
                    def failed = report.run.stats.tests.failed
                    def passed = total - failed
                    def passRate = (passed / total) * 100

                    echo "✅ Passed: ${passed} / ${total} (${passRate.toInteger()}%)"

                    if (passRate >= THRESHOLD.toInteger()) {
                        echo "✅ Pass rate is above threshold (${THRESHOLD}%)"
                    } else {
                        currentBuild.result = 'FAILURE'
                        error("❌ Pass rate (${passRate.toInteger()}%) is below threshold (${THRESHOLD}%)")
                    }
                }
            }
        }
		
        stage('Email Report') {
            steps {
                emailext(
                    subject: "${EMAIL_SUBJECT}",
                    body: """Test execution completed.<br>
                             <b>Build:</b> ${env.BUILD_URL}<br>
                             <b>Result:</b> ${currentBuild.currentResult}<br><br>
                             See attached HTML report.""",
                    to: "${RECIPIENTS}",
                    mimeType: 'text/html',
                    attachmentsPattern: 'reports/report.html',
                    attachLog: true
                )
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'reports/*.html', fingerprint: true
            publishHTML(target: [
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'reports',
                reportFiles: 'report.html',
                reportName: 'Postman API Test Report'
            ])
        }
		
		failure {
            mail to: "${RECIPIENTS}",
                 subject: "❌ FAILED: ${EMAIL_SUBJECT}",
                 body: "The Postman test pipeline failed. Check Jenkins for full logs."
        }
    }
}