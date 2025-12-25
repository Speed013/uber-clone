pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = credentials('account_id')
        AWS_DEFAULT_REGION = "us-east-1"
        AWS_ACCESS_KEY_ID = credentials('aws_access_key_id')
        AWS_SECRET_ACCESS_KEY = credentials('aws_secret_access_key')
    }

    stages {

        stage('Run Terrascan') {
            steps {
                script {
                    sh '''
                        docker run --rm \
                        -v $WORKSPACE:/iac \
                        tenable/terrascan:latest scan \
                        -d /iac/EKS_Terraform \
                        -o json > terrascan_output.json || true
                    '''

                    archiveArtifacts artifacts: 'terrascan_output.json', allowEmptyArchive: true

                    if (!fileExists('terrascan_output.json')) {
                        error "Terrascan output file was not created"
                    }

                    def jsonContent = readFile('terrascan_output.json').trim()
                    if (!jsonContent) {
                        error "Terrascan output is empty"
                    }

                    def parsedJSON = new groovy.json.JsonSlurper().parseText(jsonContent)

                    def violations = parsedJSON?.results?.violations ?: []

                    def mediumViolations = violations.findAll { it.severity == 'MEDIUM' }.size()
                    def highViolations = violations.findAll { it.severity == 'HIGH' }.size()

                    if (mediumViolations > 0 || highViolations > 0) {
                        error "Terrascan found ${mediumViolations} medium and ${highViolations} high severity vulnerabilities"
                    } else {
                        echo "Terrascan passed. No medium or high severity issues found."
                    }
                }
            }
        }

        stage('Terraform Init') {
            steps {
                sh '''
                    cd $WORKSPACE/EKS_Terraform
                    terraform init
                '''
            }
        }

        stage('Terraform Validate') {
            steps {
                sh '''
                    cd $WORKSPACE/EKS_Terraform
                    terraform validate
                '''
            }
        }

        stage('Terraform Apply') {
            steps {
                sh '''
                    cd $WORKSPACE/EKS_Terraform
                    terraform apply -auto-approve
                '''
            }
        }
    }

    post {
        always {
            echo 'Pipeline execution completed.'
        }
        failure {
            echo 'Pipeline failed. Please check the logs for details.'
        }
    }
}
