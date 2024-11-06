pipeline {
    agent any

    options {
        gitLabConnection('gitlab')
    }

    stages {
        stage("build") {
            agent {
                docker {
                    image 'python:3.6'
                    args '-u root'
                }
            }
            steps {
                sh """
                pip3 install --user virtualenv
                python3 -m virtualenv env
                . env/bin/activate
                pip3 install -r requirements.txt
                python3 manage.py check
                """
            }
        }
        stage("test") {
            agent {
                docker {
                    image 'python:3.6'
                    args '-u root'
                }
            }
            steps {
                sh """
                pip3 install --user virtualenv
                python3 -m virtualenv env
                . env/bin/activate
                pip3 install -r requirements.txt
                python3 manage.py test taskManager
                """
            }
        }
        stage("sast") {
            steps {
                withCredentials([string(credentialsId: 'dojo-host', variable: 'DOJO_HOST'), string(credentialsId: 'dojo-api-token', variable: 'DOJO_API_TOKEN')]) {
                    sh """
                    docker run -v \$(pwd):/src --rm hysnsec/bandit -r /src -f json -o /src/bandit-output.json || true       # ignore exit code 1, because the next command doesn't execute when the previous command failed
                    python3 upload-results.py --host $DOJO_HOST --api_key $DOJO_API_TOKEN --engagement_id 1 --product_id 1 --lead_id 1 --environment "Production" --result_file bandit-output.json --scanner 'Bandit Scan'
                    """
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'bandit-output.json', fingerprint: true
                }
            }
        }
        stage("integration") {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    echo "This is an integration step."
                    sh "exit 1"
                }
            }
        }
        stage("zap-baseline") {
            steps {
                withCredentials([string(credentialsId: 'dojo-host', variable: 'DOJO_HOST'), string(credentialsId: 'dojo-api-token', variable: 'DOJO_API_TOKEN')]) {
                    sh """
                    docker run --user \$(id -u):\$(id -g) -w /zap -v \$(pwd):/zap/wrk:rw --rm softwaresecurityproject/zap-stable:2.14.0 zap-baseline.py -t https://prod-g8p07dq6.lab.practical-devsecops.training/ -d -x zap-output.xml
                    python3 upload-results.py --host $DOJO_HOST --api_key $DOJO_API_TOKEN --engagement_id 1 --product_id 1 --lead_id 1 --environment "Production" --result_file zap-output.xml --scanner 'ZAP Scan'
                    """
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'bandit-output.json', fingerprint: true
                }
            }
        }
        stage("prod") {
            steps {
                timeout(time: 10, unit: 'SECONDS') {
                    input "Deploy to production?"
                }
                echo "This is a deploy step."
            }
        }
    }
    post {
        failure {
            updateGitlabCommitStatus(name: "\${env.STAGE_NAME}", state: 'failed')
        }
        unstable {
            updateGitlabCommitStatus(name: "\${env.STAGE_NAME}", state: 'failed')
        }
        success {
            updateGitlabCommitStatus(name: "\${env.STAGE_NAME}", state: 'success')
        }
        aborted {
            updateGitlabCommitStatus(name: "\${env.STAGE_NAME}", state: 'canceled')
        }
        always {
            deleteDir()                     // clean up workspace
            dir("${WORKSPACE}@tmp") {       // clean up tmp directory
                deleteDir()
            }
            dir("${WORKSPACE}@script") {    // clean up script directory
                deleteDir()
            }
        }
    }
}
