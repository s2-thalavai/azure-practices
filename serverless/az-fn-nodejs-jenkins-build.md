
# Azure Function JEnkinsfile

## simple version

```groovy

pipeline {
    agent any	
	
    environment {
        RESOURCE_GROUP = "Zone-RG"
        FUNCTION_APP   = "platform-svc-fn-app"
    }

    stages {
        stage('Checkout Code') {
            steps {
                dir('platform-services-fn-app') {
                    checkout scmGit(
                        branches: [[name: '*/${BRANCH_TO_BUILD}']],
                        userRemoteConfigs: [[
                            credentialsId: 'is-apps-ssh-key',
                            url: 'git@github.com:org/platform-services-fn-app.git'
                        ]]
                    )
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                dir('platform-services-fn-app') {
                    powershell '''
                        npm ci --production
                    '''
                }
            }
        }

        stage('Azure Login') {
            steps {
                    powershell '''
                        az login --service-principal --username 1234-210129812-211212-1221 --password 1211221~12121212~11212 --tenant 1221121212
                        az account set --subscription 12121221-212112-4e3c-12122-12121221
                    '''
            }
        }

        stage('Publish Functions') {
            steps {
                dir('platform-services-fn-app') {
                    powershell '''
                        npm install -g azure-functions-core-tools@4 --unsafe-perm true
                        func azure functionapp publish $env:FUNCTION_APP --javascript --nozip
                    '''
                }
            }
        }
    }
}

```


-------


## 

```groovy
pipeline {
    agent any

    parameters {
        string(name: 'BRANCH_TO_BUILD', defaultValue: 'Dev-branch', description: 'Git branch to build and deploy')
        string(name: 'FUNCTION_APP', defaultValue: 'platform-svc-fn-app-nonprod', description: 'Azure Function App name')
        string(name: 'ROLLBACK_BUILD', defaultValue: '', description: 'Optional: Jenkins build number to rollback to')
    }

    environment {
        RESOURCE_GROUP    = "Zone-RG"
        STORAGE_ACCOUNT   = "docustorageqa"
        STORAGE_CONTAINER = "platform-svc-fn-app-nonprod-build"
        FUNCTION_APP_ENV  = "${params.FUNCTION_APP}"
        BRANCH_TO_BUILD_ENV = "${params.BRANCH_TO_BUILD}"
        ROLLBACK_BUILD_ENV  = "${params.ROLLBACK_BUILD}"
    }

    stages {
        stage('Checkout Code') {
            steps {
                dir('platform-services-fn-app') {
                    checkout scmGit(
                        branches: [[name: "*/${params.BRANCH_TO_BUILD}"]],
                        userRemoteConfigs: [[
                            credentialsId: 'is-apps-ssh-key',
                            url: 'git@github.com:Marlabs-Innovations-Private-Limited/platform-services-fn-app.git'
                        ]]
                    )
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                dir('platform-services-fn-app') {
                    powershell 'npm ci --production'
                }
            }
        }

        stage('Azure Login') {
            steps {
                powershell '''
                    az login --service-principal --username 1234-210129812-211212-1221 --password 1211221~12121212~11212 --tenant 1221121212
                    az account set --subscription 12121221-212112-4e3c-12122-12121221
                '''
            }
        }

        stage('Publish Functions') {
            when {
                expression { return !params.ROLLBACK_BUILD?.trim() }
            }
            steps {
                dir('platform-services-fn-app') {
                    powershell '''
                        npm install -g azure-functions-core-tools@4 --unsafe-perm true
                        func azure functionapp publish $env:FUNCTION_APP_ENV --javascript --nozip
                    '''
                }
            }
        }

        stage('Archive Artifact') {
            when {
                expression { return !params.ROLLBACK_BUILD?.trim() }
            }
            steps {
                archiveArtifacts artifacts: 'platform-services-fn-app/function.zip', fingerprint: true
            }
        }

        stage('Upload to Blob Storage') {
            when {
                expression { return !params.ROLLBACK_BUILD?.trim() }
            }
            steps {
                powershell '''
                    $artifactName = "function-$env:BUILD_NUMBER.zip"
                    az storage blob upload `
                        --account-name $env:STORAGE_ACCOUNT `
                        --container-name $env:STORAGE_CONTAINER `
                        --file "$env:WORKSPACE\\platform-services-fn-app\\function.zip" `
                        --name $artifactName `
                        --overwrite
                '''
            }
        }

        stage('List Artifacts') {
            steps {
                powershell '''
                    Write-Host "Available artifacts in $env:STORAGE_CONTAINER:"
                    az storage blob list `
                        --account-name $env:STORAGE_ACCOUNT `
                        --container-name $env:STORAGE_CONTAINER `
                        --output table
                '''
            }
        }

        stage('Rollback') {
            when {
                expression { return params.ROLLBACK_BUILD?.trim() }
            }
            steps {
                powershell '''
                    $artifactName = "function-$env:ROLLBACK_BUILD_ENV.zip"
                    Write-Host "Rolling back to artifact: $artifactName"
                    az functionapp deployment source config-zip `
                        --resource-group $env:RESOURCE_GROUP `
                        --name $env:FUNCTION_APP_ENV `
                        --src "https://$env:STORAGE_ACCOUNT.blob.core.windows.net/$env:STORAGE_CONTAINER/$artifactName"
                '''
            }
        }
    }
}

```

----------
