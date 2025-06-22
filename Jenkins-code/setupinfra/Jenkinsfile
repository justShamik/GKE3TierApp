properties([
    parameters([
        string(
            defaultValue: 'dev',
            description: 'Deployment environment (e.g., dev, prod)',
            name: 'Environment'
        ),
        choice(
            choices: ['plan', 'apply', 'destroy'],
            description: 'Terraform action to perform',
            name: 'Terraform_Action'
        )
    ])
])

pipeline {
    agent any
    options {
        ansiColor('xterm') // Enable colored output
    }

    environment {
        // Avoid hardcoding the credential file path; rely on withCredentials
        TF_VAR_service_account_key = credentials('gcp-service-account-json') // Bind credential to Terraform variable
    }

    stages {
        stage('Checkout') {
            steps {
                // Checkout Terraform code from GitHub
                git branch: 'master', credentialsId: 'github-credentials-id', url: 'https://github.com/justShamik/GKE3TierTerra.git'
            }
        }

        stage('Install Terraform') {
            steps {
                // Use tfenv for flexible Terraform version management
                sh '''
                    if ! [ -x "$(command -v terraform)" ]; then
                        curl -fsSL https://github.com/tfutils/tfenv/archive/v3.0.0.tar.gz | tar -xz
                        sudo mv tfenv-3.0.0 /usr/local/tfenv
                        sudo ln -s /usr/local/tfenv/bin/* /usr/local/bin/
                        tfenv install 1.5.0
                        tfenv use 1.5.0
                    fi
                    terraform --version
                '''
            }
        }

        stage('Initialize Terraform') {
            steps {
                // Initialize Terraform with the service account key
                withCredentials([file(credentialsId: 'gcp-service-account-json', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    sh 'terraform init'
                }
            }
        }

        stage('Terraform Plan') {
            when {
                expression { params.Terraform_Action in ['plan', 'apply'] }
            }
            steps {
                // Run Terraform plan with the service account key variable
                withCredentials([file(credentialsId: 'gcp-service-account-json', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    sh 'terraform plan -var="service_account_key=${GOOGLE_APPLICATION_CREDENTIALS}" -out=tfplan'
                }
            }
        }

        stage('Terraform Apply or Destroy') {
            when {
                expression { params.Terraform_Action in ['apply', 'destroy'] }
            }
            steps {
                // Add manual approval for apply or destroy
                script {
                    if (params.Terraform_Action in ['apply', 'destroy']) {
                        input message: "Confirm ${params.Terraform_Action} action for environment ${params.Environment}?", ok: 'Proceed'
                    }
                }
                // Apply or destroy with the service account key
                withCredentials([file(credentialsId: 'gcp-service-account-json', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    script {
                        if (params.Terraform_Action == 'apply') {
                            sh 'terraform apply tfplan '
                        } else if (params.Terraform_Action == 'destroy') {
                            sh 'terraform destroy -var="service_account_key=${GOOGLE_APPLICATION_CREDENTIALS}" -auto-approve'
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            // Cleanup workspace and remove sensitive files
            cleanWs()
            sh 'rm -f tfplan' // Remove plan file
        }
        failure {
            echo 'Terraform operation failed. Check logs for details.'
        }
    }
}
