pipeline {
    agent any

    environment {
        DOTNET_CLI_HOME = "C:\\Program Files\\dotnet"
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
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Build Docker image from the Dockerfile
                    bat "docker build -t my-dotnet-app ."
                }
            }
        }

        stage('Run Docker Container') {
            steps {
                script {
                    // Running Docker container
                    bat "docker run -d --name my-dotnet-container -p 8080:80 my-dotnet-app"
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'coreuser', passwordVariable: 'CREDENTIAL_PASSWORD', usernameVariable: 'CREDENTIAL_USERNAME')]) {
                        powershell '''
                        
                        $credentials = New-Object System.Management.Automation.PSCredential($env:CREDENTIAL_USERNAME, (ConvertTo-SecureString $env:CREDENTIAL_PASSWORD -AsPlainText -Force))

                        
                        New-PSDrive -Name X -PSProvider FileSystem -Root "\\\\LAPTOP-DFRQ3ILG\\coreapp" -Persist -Credential $credentials

                        
                        Copy-Item -Path '.\\publish\\*' -Destination 'X:\\' -Force

                        
                        Remove-PSDrive -Name X
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Build, test, publish, and deployment successful!'
        }
    }
}
