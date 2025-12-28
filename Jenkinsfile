pipeline {
    agent any

    triggers {
        githubPush()
        pollSCM('H/5 * * * *')
    }

    environment {
        REMOTE_HOST = "${env.EC2_INFRA_IP}"
        REMOTE_USER = "ec2-user"
        REMOTE_DIR = "/home/ec2-user/microservices/discovery-server"
        SSH_CREDENTIALS_ID = "ec2-ssh-key"
    }

    tools {
        maven 'Maven 3'
        jdk 'JDK 17'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                bat 'mvn clean package -DskipTests'
            }
        }

        stage('Deploy to EC2') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'SSH_KEY')]) {
                    powershell '''
                        $ErrorActionPreference = "Stop"
                        Write-Host "--- Start Deployment ---"

                        # 1. Fix Key Permissions
                        Write-Host "Fixing key permissions..."
                        $keyPath = $env:SSH_KEY
                        $acl = Get-Acl $keyPath
                        $acl.SetAccessRuleProtection($true, $false)
                        $currentUser = [System.Security.Principal.WindowsIdentity]::GetCurrent().Name
                        $rule = New-Object System.Security.AccessControl.FileSystemAccessRule($currentUser, "FullControl", "Allow")
                        $acl.SetAccessRule($rule)
                        Set-Acl $keyPath $acl

                        # 2. Deployment Steps
                        $remote = "$env:REMOTE_USER@$env:REMOTE_HOST"
                        
                        Write-Host "Stopping existing service on $remote..."
                        # Added ConnectTimeout to fail fast if network is down
                        ssh -i $keyPath -o StrictHostKeyChecking=no -o ConnectTimeout=10 $remote "pkill -f discovery-server; exit 0"
                        Write-Host "Service stopped."

                        Write-Host "Creating directory $env:REMOTE_DIR..."
                        ssh -i $keyPath -o StrictHostKeyChecking=no -o ConnectTimeout=10 $remote "mkdir -p $env:REMOTE_DIR"
                        
                        Write-Host "Uploading JAR..."
                        $jarFiles = @(Get-Item "target/*.jar")
                        $jarFile = $jarFiles[0]
                        Write-Host "Found JAR: $jarFile"
                        scp -i $keyPath -o StrictHostKeyChecking=no -o ConnectTimeout=10 $jarFile $remote":"$env:REMOTE_DIR/discovery-server.jar
                        Write-Host "Upload complete."
                        
                        Write-Host "Starting service..."
                        # Added sleep 2 to ensure nohup detaches before SSH closes
                        ssh -i $keyPath -o StrictHostKeyChecking=no -o ConnectTimeout=10 $remote "nohup java -jar $env:REMOTE_DIR/discovery-server.jar > $env:REMOTE_DIR/log.txt 2>&1 & sleep 2"
                        Write-Host "Service start command sent."
                        
                        Write-Host "--- Deployment Complete ---"
                    '''
                }
            }
        }
    }
}
