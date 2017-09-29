pipeline {
    agent {
        label "docker"
    }
    environment {
        REL_VERSION = "${BRANCH_NAME.contains('release-') ? BRANCH_NAME.drop(BRANCH_NAME.lastIndexOf('-')+1) + '.' + BUILD_NUMBER : 'M.' + BUILD_NUMBER}"
        ACR_CREDS = credentials("acr")
    }

    options {
        buildDiscarder(logRotator(numToKeepStr:'20'))
        timestamps()
        skipStagesAfterUnstable()
    }

    stages {
        stage('Build Backend') {
            agent {
                docker {
                    image 'maven:3-alpine'
                    args '-v /root/.m2:/root/.m2'
                    reuseNode true
                }
            }
            steps {
                sh './mvnw -B clean install'
            }
            post {
                always {
                    junit '**/surefire-reports/**/*.xml'

                }
            }
        }
        stage('Additional Tests') {
            parallel {
                stage('Frontend') {
                    when {
                        anyOf {
                            branch "master"
                            branch "release-*"
                            changeset "src/main/webapp/**/*"
                        }
                    }
                    agent {
                        dockerfile {
                            args "-v /tmp:/tmp"
                            dir "docker/gulp"
                            reuseNode true
                        }
                    }
                    steps {
                        sh 'yarn install'
                        sh 'yarn add gulp-cli'
                        sh 'gulp test'
                    }
                    post {
                        always {
                            junit 'target/test-reports/karma/TESTS-results.xml'
                        }
                    }
                }
                stage('Performance') {
                    when {
                        anyOf {
                            branch "master"
                            branch "release-*"
                        }
                    }
                    agent {
                        docker {
                            image 'maven:3-alpine'
                            args '-v $HOME/.m2:/root/.m2'
                            reuseNode true
                        }
                    }
                    steps {
                        sh 'mvn -B gatling:execute'
                    }
                    post {
                        always {
                            gatlingArchive()
                        }
                    }
                }
            }
        }
        stage('Build Container') {
            steps {
                sh "docker login -u ${ACR_CREDS_USR} -p ${ACR_CREDS_PSW} pipelineregistry.azurecr.io"
                sh "./mvnw -B docker:build -Ddocker-tag=${BUILD_ID} -DpushImageTag"
            }
        }
        stage('Deploy to Staging') {
            steps {
                acsDeploy azureCredentialsId: 'acs-staging', configFilePaths: 'acsK8sStaging.yml',
                    containerRegistryCredentials: [[credentialsId: 'acr', url: 'https://pipelineregistry.azurecr.io']],
                    containerService: 'abayerPipelineDemo | Kubernetes', enableConfigSubstitution: true,
                    resourceGroupName: 'abayer-pipeline-demo-acs', sshCredentialsId: 'acs-staging-creds'
                sh "docker login -u ${ACR_CREDS_USR} -p ${ACR_CREDS_PSW} pipelineregistry.azurecr.io"
                sh "docker pull pipelineregistry.azurecr.io/jhipstersampleapplication:${BUILD_NUMBER}"
                sh "docker tag pipelineregistry.azurecr.io/jhipstersampleapplication:${BUILD_NUMBER} pipelineregistry.azurecr.io/jhipstersampleapplication:latest"
                sh "docker push pipelineregistry.azurecr.io/jhipstersampleapplication:latest"
            }
        }
        stage('Deploy to production') {
            steps {
                input message: 'Deploy to production?', ok: 'Fire zee missiles!'
                acsDeploy azureCredentialsId: 'acs-staging', configFilePaths: 'acsK8sProduction.yml',
                    containerRegistryCredentials: [[credentialsId: 'acr', url: 'https://pipelineregistry.azurecr.io']],
                    containerService: 'abayerPipelineDemo | Kubernetes', enableConfigSubstitution: true,
                    resourceGroupName: 'abayer-pipeline-demo-acs', sshCredentialsId: 'acs-staging-creds'
            }
        }
    }
}
