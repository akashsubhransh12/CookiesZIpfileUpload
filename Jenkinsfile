pipeline {
    agent any

    /*
     * PARAMETERS
     * PRODUCT   → Target application   Akash Subhransh
     * UPLOAD_ZIP → ZIP/DLL file uploaded from local system
     */
    parameters {
        choice(
            name: 'PRODUCT',
            choices: ['Agoda', 'Booking'],
            description: 'Select product'
        )

        // Replaces FILE_NAME string parameter
        stashedFile(
            name: 'UPLOAD_ZIP',
            description: 'Upload a ZIP file containing product files'
        )
    }

    /*
     * PATH CONFIGURATION Akash Subhransh
     */
    environment {
        LOCAL_BASE  = 'D:\\TravelApp\\Cookies'
        REMOTE_BASE = 'C:\\TravelApp\\Cookies'
    }

    stages {

        /*
         * Resolve server IP and credential ID Akash Subhransh
         */
        stage('Resolve Server') {
            steps {
                script {
                    def servers = [
                        'Agoda'  : [ip: '10.10.10.54', cred: 'agoda-creds'],
                        'Booking': [ip: '10.10.10.63', cred: 'booking-creds']
                    ]

                    if (!servers.containsKey(params.PRODUCT)) {
                        error "Server not configured for ${params.PRODUCT}"
                    }

                    env.SERVER_IP = servers[params.PRODUCT].ip
                    env.CREDS_ID  = servers[params.PRODUCT].cred
                }
            }
        }

        /*
         * Prepare Uploaded ZIP File Akash Subhransh
         */
        stage('Store File Locally') {
            steps {
                script {
                    // Unstash uploaded file
                    unstash 'UPLOAD_ZIP'

                    // Ensure product folder exists
                    def localProductFolder = "${env.LOCAL_BASE}\\${params.PRODUCT}"
                    bat """
                        if not exist "${localProductFolder}" mkdir "${localProductFolder}"
                    """

                    // Rename file to original filename
                    bat """
                        ren UPLOAD_ZIP %UPLOAD_ZIP_FILENAME%
                    """

                    // Construct full local file path
                    def localFilePath = "${localProductFolder}\\${env.UPLOAD_ZIP_FILENAME}"
                    env.LOCAL_FILE_PATH = localFilePath

                    echo "Uploaded ZIP stored locally at: ${localFilePath}"
                }
            }
        }

        /*
         * Deploy file to target server
         */
        stage('Deploy to Target Server') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: env.CREDS_ID,
                        usernameVariable: 'DEPLOY_USER',
                        passwordVariable: 'DEPLOY_PASS'
                    )
                ]) {
                    powershell """
                    # Create credential
                    \$secPass = ConvertTo-SecureString \$env:DEPLOY_PASS -AsPlainText -Force
                    \$cred = New-Object System.Management.Automation.PSCredential(\$env:DEPLOY_USER, \$secPass)

                    # Variables
                    \$server    = '${env.SERVER_IP}'
                    \$product   = '${params.PRODUCT}'
                    \$fileName  = '${env.UPLOAD_ZIP_FILENAME}'
                    \$localFile = '${env.LOCAL_FILE_PATH}'
                    \$remoteDir = '${REMOTE_BASE}\\\\' + \$product
                    \$backupDir = \$remoteDir + '-backup-' + (Get-Date -Format 'yyyyMMdd_HHmmss')

                    # Create remote session
                    \$session = New-PSSession -ComputerName \$server -Credential \$cred

                    # Prepare remote directory & backup existing files
                    Invoke-Command -Session \$session -ScriptBlock {
                        param(\$remoteDir, \$backupDir, \$fileName)

                        if (!(Test-Path \$remoteDir)) {
                            New-Item -ItemType Directory -Path \$remoteDir -Force | Out-Null
                        }

                        if (Test-Path "\$remoteDir\\\\\$fileName") {
                            New-Item -ItemType Directory -Path \$backupDir -Force | Out-Null
                            Copy-Item "\$remoteDir\\\\\$fileName" \$backupDir -Force
                            Remove-Item "\$remoteDir\\\\\$fileName" -Force
                        }
                    } -ArgumentList \$remoteDir, \$backupDir, \$fileName

                    # Copy file to server
                    Copy-Item -Path \$localFile -Destination \$remoteDir -ToSession \$session -Force

                    # Close session
                    Remove-PSSession \$session
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Deployment completed successfully"
        }
        failure {
            echo "Deployment failed"
        }
    }
}
