// Helper functions
def installTerraform(String version) {
    def installScript = """
        wget https://releases.hashicorp.com/terraform/\${version}/terraform_\${version}_linux_amd64.zip || exit 1
        unzip terraform_\${version}_linux_amd64.zip || exit 1
        sudo mv terraform /usr/local/bin/ || exit 1
        rm terraform_\${version}_linux_amd64.zip || exit 1
    """
    
    def result = sh(script: installScript, returnStatus: true)
    if (result != 0) {
        error "Failed to install Terraform version \${version}"
    }
}

def verifyTerraformInstallation(String expectedVersion) {
    def tfVersion = sh(
        script: 'terraform --version',
        returnStdout: true
    ).trim()
    
    if (!tfVersion.contains(expectedVersion)) {
        error "Terraform version mismatch. Expected \${expectedVersion}, but found \${tfVersion}"
    }
    return tfVersion
}

def getAWSCredentials() {
    def vaultToken = sh(
        script: """
            curl -s -X POST \
            -d '{"role_id":"${ROLE_ID}", "secret_id":"${SECRET_ID}"}' \
            ${VAULT_ADDR}/v1/auth/approle/login | jq -r '.auth.client_token'
        """,
        returnStdout: true
    ).trim()
    
    def awsCredentials = sh(
        script: """
            curl -s -H "X-Vault-Token: ${vaultToken}" \
            ${VAULT_ADDR}/v1/${VAULT_PATH} | jq -r '.data.data'
        """,
        returnStdout: true
    ).trim()
    
    return readJSON(text: awsCredentials)
}

pipeline {
    agent any
    
    environment {
        TF_PATH = "${WORKSPACE}/terraform"
        TERRAFORM_VERSION = '1.5.7'
        VAULT_ADDR = 'https://vault.example.com'
        VAULT_PATH = 'kv/data/aws'
        ROLE_ID = credentials('vault-role-id')
        SECRET_ID = credentials('vault-secret-id')
    }
    
    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: 'Select environment')
        choice(name: 'ACTION', choices: ['plan', 'apply', 'destroy'], description: 'Terraform action')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Setup Terraform') {
            steps {
                script {
                    try {
                        // Check if Terraform is installed
                        def tfInstalled = sh(
                            script: 'command -v terraform',
                            returnStatus: true
                        ) == 0

                        if (!tfInstalled) {
                            echo "Terraform not found. Installing Terraform version ${TERRAFORM_VERSION}..."
                            installTerraform(TERRAFORM_VERSION)
                            def version = verifyTerraformInstallation(TERRAFORM_VERSION)
                            echo "Terraform installed successfully: ${version}"
                        } else {
                            // Verify version of installed Terraform
                            def currentVersion = sh(
                                script: 'terraform --version',
                                returnStdout: true
                            ).trim()

                            if (!currentVersion.contains(TERRAFORM_VERSION)) {
                                echo "Updating Terraform to version ${TERRAFORM_VERSION}..."
                                installTerraform(TERRAFORM_VERSION)
                                def version = verifyTerraformInstallation(TERRAFORM_VERSION)
                                echo "Terraform updated successfully: ${version}"
                            } else {
                                echo "Correct Terraform version already installed: ${currentVersion}"
                            }
                        }

                        // Set up Terraform environment
                        def tfHome = tool name: 'Terraform', type: 'terraform'
                        env.PATH = "${tfHome}:${env.PATH}"
                    } catch (Exception e) {
                        error "Failed to set up Terraform: ${e.getMessage()}"
                    }
                }
            }
        }

        stage('Terraform Init & Workspace') {
            steps {
                dir(TF_PATH) {
                    script {
                        def awsCreds = getAWSCredentials()
                        withEnv([
                            "AWS_ACCESS_KEY_ID=${awsCreds.access_key}",
                            "AWS_SECRET_ACCESS_KEY=${awsCreds.secret_key}"
                        ]) {
                            sh """
                                terraform init \
                                -backend-config="bucket=company-terraform-states" \
                                -backend-config="key=${params.ENVIRONMENT}/infrastructure.tfstate" \
                                -backend-config="region=us-west-2" \
                                -backend-config="encrypt=true"
                                
                                terraform workspace select ${params.ENVIRONMENT} || terraform workspace new ${params.ENVIRONMENT}
                            """
                        }
                    }
                }
            }
        }

        stage('Terraform Format') {
            steps {
                dir(TF_PATH) {
                    sh 'terraform fmt -check'
                }
            }
        }

        stage('Terraform Validate') {
            steps {
                dir(TF_PATH) {
                    script {
                        def awsCreds = getAWSCredentials()
                        withEnv([
                            "AWS_ACCESS_KEY_ID=${awsCreds.access_key}",
                            "AWS_SECRET_ACCESS_KEY=${awsCreds.secret_key}"
                        ]) {
                            sh 'terraform validate'
                        }
                    }
                }
            }
        }

        stage('Terraform Plan') {
            steps {
                dir(TF_PATH) {
                    script {
                        def awsCreds = getAWSCredentials()
                        withEnv([
                            "AWS_ACCESS_KEY_ID=${awsCreds.access_key}",
                            "AWS_SECRET_ACCESS_KEY=${awsCreds.secret_key}"
                        ]) {
                            sh """
                                terraform plan \
                                -var-file="environments/${params.ENVIRONMENT}.tfvars" \
                                -out=tfplan
                            """
                        }
                    }
                }
            }
        }

        stage('Apply Approval') {
            when {
                expression { 
                    return params.ACTION == 'apply' && params.ENVIRONMENT == 'prod'
                }
            }
            steps {
                input message: 'Apply production changes?'
            }
        }

        stage('Terraform Apply') {
            when {
                expression { params.ACTION == 'apply' }
            }
            steps {
                dir(TF_PATH) {
                    script {
                        def awsCreds = getAWSCredentials()
                        withEnv([
                            "AWS_ACCESS_KEY_ID=${awsCreds.access_key}",
                            "AWS_SECRET_ACCESS_KEY=${awsCreds.secret_key}"
                        ]) {
                            sh 'terraform apply -auto-approve tfplan'
                        }
                    }
                }
            }
        }

        stage('Terraform Destroy') {
            when {
                expression { params.ACTION == 'destroy' }
            }
            steps {
                dir(TF_PATH) {
                    script {
                        if (params.ENVIRONMENT == 'prod') {
                            input message: 'Are you sure you want to destroy production infrastructure?'
                        }
                        def awsCreds = getAWSCredentials()
                        withEnv([
                            "AWS_ACCESS_KEY_ID=${awsCreds.access_key}",
                            "AWS_SECRET_ACCESS_KEY=${awsCreds.secret_key}"
                        ]) {
                            sh """
                                terraform destroy \
                                -var-file="environments/${params.ENVIRONMENT}.tfvars" \
                                -auto-approve
                            """
                        }
                    }
                }
            }
        }

        stage('Show Outputs') {
            when {
                expression { params.ACTION == 'apply' }
            }
            steps {
                dir(TF_PATH) {
                    script {
                        def awsCreds = getAWSCredentials()
                        withEnv([
                            "AWS_ACCESS_KEY_ID=${awsCreds.access_key}",
                            "AWS_SECRET_ACCESS_KEY=${awsCreds.secret_key}"
                        ]) {
                            def outputs = sh(
                                script: 'terraform output -json',
                                returnStdout: true
                            ).trim()
                            
                            echo "Terraform Outputs: ${outputs}"
                            def outputsMap = readJSON text: outputs
                            echo "Instance ID: ${outputsMap.instance_id.value}"
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline executed successfully!'
        }
        failure {
            echo 'Pipeline execution failed!'
        }
    }
}
