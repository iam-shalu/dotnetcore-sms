pipeline {
    agent any

    environment {
        DOTNET_CLI_HOME = "C:\\Program Files\\dotnet"
        AZURE_SUBSCRIPTION_ID = '<your-subscription-id>'
        AZURE_CLIENT_ID = '<your-client-id>'
        AZURE_CLIENT_SECRET = '<your-client-secret>'
        AZURE_TENANT_ID = '<your-tenant-id>'
        AZURE_RESOURCE_GROUP = '<your-resource-group>'
        AZURE_VM_NAME = '<your-vm-name>'
        AZURE_VM_IP = '<your-vm-ip>'
        AZURE_VM_USER = '<your-vm-username>'
        AZURE_VM_PASSWORD = credentials('your-vm-password-id')  // Store the VM password in Jenkins credentials
        AZURE_STORAGE_ACCOUNT = '<your-storage-account>'
        AZURE_STORAGE_KEY = credentials('your-storage-key-id')  // Store the storage key in Jenkins credentials
        AZURE_CONTAINER_NAME = '<your-container-name>'
        APPLICATION_ZIP = 'application.zip'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                script {
                    // Restoring dependencies
                    bat "dotnet restore"

                    // Building the application
                    bat "dotnet build --configuration Release"
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    // Running tests
                    bat "dotnet test --no-restore --configuration Release"
                }
            }
        }

        stage('Publish') {
            steps {
                script {
                    // Publishing the application
                    bat "dotnet publish --no-restore --configuration Release --output .\\publish"

                    // Compress the published output into a zip file
                    bat "powershell Compress-Archive -Path .\\publish\\* -DestinationPath .\\${APPLICATION_ZIP}"
                }
            }
        }

        stage('Azure Login') {
            steps {
                script {
                    // Login to Azure
                    bat 'az login --service-principal -u %AZURE_CLIENT_ID% -p %AZURE_CLIENT_SECRET% --tenant %AZURE_TENANT_ID%'
                    bat 'az account set --subscription %AZURE_SUBSCRIPTION_ID%'
                }
            }
        }

        stage('Upload to Azure Storage') {
            steps {
                script {
                    // Upload the application package to Azure Storage
                    bat 'az storage blob upload --account-name %AZURE_STORAGE_ACCOUNT% --account-key %AZURE_STORAGE_KEY% --container-name %AZURE_CONTAINER_NAME% --file %APPLICATION_ZIP% --name %APPLICATION_ZIP%'
                }
            }
        }

        stage('Deploy to Azure VM') {
            steps {
                script {
                    // Generate a SAS token for the storage blob
                    def sasToken = bat(script: """
                        az storage blob generate-sas --account-name %AZURE_STORAGE_ACCOUNT% --account-key %AZURE_STORAGE_KEY% --container-name %AZURE_CONTAINER_NAME% --name %APPLICATION_ZIP% --permissions r --expiry `date -u -d "1 hour" '+%Y-%m-%dT%H:%MZ'` -o tsv
                    """, returnStdout: true).trim()

                    // Construct the full URL to the application package
                    def blobUrl = "https://${AZURE_STORAGE_ACCOUNT}.blob.core.windows.net/${AZURE_CONTAINER_NAME}/${APPLICATION_ZIP}?${sasToken}"

                    // Use Azure CLI to run commands on the VM to download and deploy the application
                    bat """
                        az vm run-command invoke -g %AZURE_RESOURCE_GROUP% -n %AZURE_VM_NAME% --command-id RunPowerShellScript --scripts @'
                        \$storageUrl = "${blobUrl}"
                        \$destinationPath = "C:\\path\\to\\destination\\${APPLICATION_ZIP}"
                        Invoke-WebRequest -Uri \$storageUrl -OutFile \$destinationPath
                        Expand-Archive -Path \$destinationPath -DestinationPath "C:\\path\\to\\destination"
                        # Add any other deployment commands here, e.g., starting services, configuring the application, etc.
                        '@
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'Build, test, publish, and deploy successful!'
        }
    }
}
