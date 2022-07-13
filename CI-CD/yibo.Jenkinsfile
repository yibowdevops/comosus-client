pipeline {
    agent any

    options {
        // Keep artifacts and logs of last 10 builds
        buildDiscarder(logRotator(numToKeepStr: '10'))
        // Disallows concurrent executions of the Pipeline.
        disableConcurrentBuilds()
        // Sets a timeout period for the Pipeline run, after which Jenkins should abort the Pipeline.
        // Set timeout period as 1 hour  to prevent infinite blocks in our Jenkins.
        timeout(time: 1, unit: 'HOURS')
        // Prepends all console output generated by the Pipeline run with the time at which the line was emitted.
        timestamps()
    }

    tools {
        dockerTool 'docker-latest'
    }

    environment {
        IMAGE_VERSION = "${env.BUILD_ID}"
        AWS_REGION = 'ap-southeast-2'
        CYPRESS_RECORD_KEY = credentials('cypress-record-key')
        N_G_ENDPOINT = credentials('N_G_ENDPOINT')
        ECR_URL = credentials('ECR_REGISTRY_URL')
        AWS_ECR_URL = credentials('AWS_ECR_URL')
    }



    
    stages {
        stage('Cleanup Worker space') {
            steps {
                cleanWs()
                checkout scm
            }
        }

        stage('Git check out') {
            steps{
                echo 'Git check out...'
                // Get source code from a GitHub repository
                git branch: env.BRANCH_NAME, url:'https://github.com/A-Comosus/comosus-client.git'
            }
        }
        stage('Install dependencies') {
            steps {
                echo 'Install dependencies...'
                sh 'yarn'
                sh 'yarn codegen'
            }
        }

        stage('Test') {
            steps {
                echo 'Testing..'
                sh 'yarn test:coverage'
                sh 'yarn e2e:start-server:record'
            }
        }
        
        stage('Checking for linter error') {
            steps {
                echo 'Checking for linter error....'
                sh 'yarn lint'
            }
        }

        stage('Build') {
            steps {
                echo 'Building....'
                sh 'yarn build:with-codegen'
            }
        }
               
        stage('Build and Upload Image to ECR') {
            
            when {
                anyOf {
                    branch 'develop'
                    branch 'CI-CD'
                }
            }

            steps {
               script{
                    docker.withRegistry("https://${AWS_ECR_URL}", 'ecr:ap-southeast-2:AWS_CREDENTIAL') {
                        app = docker.build("${ECR_URL}:${env.BUILD_NUMBER}", "--build-arg N_G_ENDPOINT=${N_G_ENDPOINT} -f Dockerfile.yibo .")
                        app.push()
                    }
                }
            } 
        }

    post {
        always {
            withCredentials([string(credentialsId: 'AWS_REPOSITORY_URL_SECRET', variable: 'AWS_ECR_URL')]) {
                deleteDir()
            }
        }
        success {  
            echo 'Pipeline Successed :)'  
        }
        failure {  
            mail to: 'Norris.wu.au@outlook.com',
                 subject: "Failed Pipeline: ${currentBuild.fullDisplayName}",
                 body: "Project: ${env.JOB_NAME}, Build Number: ${env.BUILD_NUMBER}, Something is wrong with ${env.BUILD_URL}"
        }
    }
}