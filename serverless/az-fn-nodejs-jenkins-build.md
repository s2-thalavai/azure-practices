
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
                            credentialsId: 'apps-ssh-key',
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

        stage('Publh Functions') {
            steps {
                dir('platform-services-fn-app') {
                    powershell '''
                        npm install -g azure-functions-core-tools@4 --unsafe-perm true
                        func azure functionapp publh $env:FUNCTION_APP --javascript --nozip
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
        string(name: 'BRANCH_TO_BUILD', defaultValue: 'sprint-01-Dev', description: 'Git branch to build and deploy')
        string(name: 'FUNCTION_APP', defaultValue: 'platform-svc-fn-app-nonprod', description: 'Azure Function App name')
        string(name: 'ROLLBACK_BUILD', defaultValue: '', description: 'Optional: Jenkins build number to rollback to')
    }
    environment {
        def timestamp = new Date().format("yyyyMMddHHmmss")
        BUILD_NUMBER = "${timestamp}"
        RESOURCE_GROUP = "Sandbox-Zone-RG"
        STORAGE_ACCOUNT = "appsdocustorageqa"
        STORAGE_CONTAINER = "platform-svc-fn-app-nonprod-build"
        FUNCTION_APP_ENV = "${params.FUNCTION_APP}"
    }
    stages {
        stage('Checkout Code') {
            steps {
                dir('platform-services-fn-app') {
                    checkout scmGit(
                        branches: [[name: "*/${params.BRANCH_TO_BUILD}"]],
                        userRemoteConfigs: [[
                            credentialsId: 'apps-ssh-key',
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
                    az login --service-principal --username 123-19a6-4b48-b3ce-1221 --password 1212~111~11 --tenant 1221-11-11-11-94736facc3be
                    az account set --subscription 111-11-11-11-111
                '''
            }
        }
        stage('Publh Functions') {
            when {
                expression {
                    return !params.ROLLBACK_BUILD?.trim()
                }
            }
            steps {
                dir('platform-services-fn-app') {
                    powershell """
                        npm install -g azure-functions-core-tools@4 --unsafe-perm true
        
                        # Publh directly to Azure
                        func azure functionapp publh ${env.FUNCTION_APP_ENV} --javascript --nozip
        
                        # Optionally copy local.settings.json for rollback/archive purposes
                        Copy-Item "local.settings.json" "local.settings.json.bak" -Force
        
                        # Package deployment-ready ZIP with build number
                        func pack --output "${env.WORKSPACE}\\platform-function-${env.BUILD_NUMBER}"
                    """
                }
            }
        }
     stage('Archive and Upload in Parallel') {
        parallel {       
            stage('Archive Artifact') {
                when {
                    expression {
                        return !params.ROLLBACK_BUILD?.trim()
                    }
                }
                steps {
                    archiveArtifacts artifacts: "platform-function-${env.BUILD_NUMBER}/platform-services-fn-app.zip", fingerprint: true
                }
            }
            
            stage('Upload to Blob Storage') {
                steps {
                    script {
                        def artifactName = "platform-function-${env.BUILD_NUMBER}/platform-services-fn-app.zip"
                        def key = powershell(
                            script: """
                                az storage account keys lt `
                                  --account-name appsdocustorageqa `
                                  --resource-group Sandbox-Zone-RG `
                                  --query "[0].value" -o tsv
                            """,
                            returnStdout: true
                        ).trim()
            
                        powershell """
                            az storage blob upload `
                              --auth-mode key `
                              --account-name ${env.STORAGE_ACCOUNT} `
                              --container-name ${env.STORAGE_CONTAINER} `
                              --file "${env.WORKSPACE}\\${artifactName}" `
                              --name "${artifactName}" `
                              --account-key ${key} `
                              --overwrite
                        """
                    }
                }
            }
        }
     }
    }
}

```

----------

```bash
az storage account show --name appsdocustorageqa --resource-group Zone-RG --query networkRuleSet

az storage account network-rule add --resource-group Zone-RG --account-name docustorageqa --ip-address 123.123.123.123


az vm lt-ip-addresses --name MLB-Modernization --resource-group Zone-RG --query "[].virtualMachine.network.publicIpAddresses[].ipAddress" -o tsv

az storage blob upload  --account-name "docustorageqa"  --container-name "platform-svc-fn-app-nonprod-build" --file "function-20251206060818/platform-services-fn-app.zip"  --name "platform-services-fn-app-20251206060818.zip" --sas-token "sp=" --overwrite

az storage account update --resource-group Zone-RG --name docustorageqa --default-action Allow

```

----------
