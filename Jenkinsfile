pipeline {
    agent any

    environment {
        DOTNET_CLI_HOME = "C:\\Program Files\\dotnet"
        AZURE_SUBSCRIPTION_ID = credentials('subscription-id')
        AZURE_CLIENT_ID = credentials('client-id')
        AZURE_CLIENT_SECRET = credentials('client-secret')
        AZURE_TENANT_ID = credentials('tenant-id')
        AZURE_RESOURCE_GROUP = 'dotnetapp'
        AZURE_VM_NAME = 'vmdotnetapp'
        AZURE_VM_IP = '13.71.99.138'
        AZURE_VM_USER = 'dotnetadmin'
        AZURE_VM_PASSWORD = credentials('vmpassword-id')  // Store the VM password in Jenkins credentials
        AZURE_STORAGE_ACCOUNT = 'stdotnetapp'
        AZURE_STORAGE_KEY = credentials('storage-key-id')  // Store the storage key in Jenkins credentials
        AZURE_CONTAINER_NAME = 'condotnetapp'
        APPLICATION_ZIP = 'dotnetcore-sms.zip'
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

        stage('Generate SAS Token') {
            steps {
                script {
                    // Generate SAS token for the uploaded blob
                    def expiryDate = new Date() + 1 // Expiry date 1 day from now
                    def expiryFormatted = expiryDate.format("yyyy-MM-dd'T'HH:mm:ss'Z'") // Format the expiry date
                    def sasTokenCommand = "az storage blob generate-sas --account-name %AZURE_STORAGE_ACCOUNT% --account-key %AZURE_STORAGE_KEY% --container-name %AZURE_CONTAINER_NAME% --name %APPLICATION_ZIP% --permissions r --expiry ${expiryFormatted} -o tsv"
                    def sasToken = bat(script: sasTokenCommand, returnStdout: true).trim()
                    env.SAS_TOKEN = sasToken
                }
            }
        }

        stage('Deploy to Azure VM') {
            steps {
                script {
                    // Construct URL with SAS token
                    def blobUrl = "https://${AZURE_STORAGE_ACCOUNT}.blob.core.windows.net/${AZURE_CONTAINER_NAME}/${APPLICATION_ZIP}?${env.SAS_TOKEN}"

                    // Use Azure CLI to run commands on the VM to download and deploy the application
                    bat """
                        az vm run-command invoke -g %AZURE_RESOURCE_GROUP% -n %AZURE_VM_NAME% --command-id RunPowerShellScript --scripts @'
                        \$storageUrl = "${blobUrl}"
                        \$destinationPath = "D:\\dotnetapp\\${APPLICATION_ZIP}"
                        Invoke-WebRequest -Uri \$storageUrl -OutFile \$destinationPath
                        Expand-Archive -Path \$destinationPath -DestinationPath "D:\\dotnetapp"
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
