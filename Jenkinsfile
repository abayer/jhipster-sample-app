pipeline {
    agent {
        label "docker"
    }
    environment {
        REL_VERSION = "${BRANCH_NAME}.${BUILD_NUMBER}"
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
                            junit 'target/test-results/karma/TESTS-results.xml'
                        }
                    }
                }
                stage('Performance') {
                    when {
                        anyOf {
                            branch "master"
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
                sh "./mvnw -B docker:build -Ddocker-tag=${REL_VERSION} -DpushImageTag"
            }
        }
        stage('Deploy to Staging') {
            steps {
                withEnv(["ENV_NAME=staging", "IMAGE_TAG=${REL_VERSION}"]) {
                    acsDeploy azureCredentialsId: 'acs-staging',
                        configFilePaths: 'acsK8s.yml',
                        containerRegistryCredentials: [[credentialsId: 'acr',
                                                        url: 'https://pipelineregistry.azurecr.io']],
                        containerService: 'abayerPipelineDemo | Kubernetes',
                        enableConfigSubstitution: true,
                        resourceGroupName: 'abayer-pipeline-demo-acs',
                        sshCredentialsId: 'acs-ssh-creds'
                }
            }
        }

        stage('Promote Image') {
            when {
                branch "master"
            }
            steps {
                sh "docker login -u ${ACR_CREDS_USR} -p ${ACR_CREDS_PSW} pipelineregistry.azurecr.io"
                sh "docker pull pipelineregistry.azurecr.io/jhipstersampleapplication:${REL_VERSION}"
                sh "docker tag pipelineregistry.azurecr.io/jhipstersampleapplication:${REL_VERSION} " +
                    "pipelineregistry.azurecr.io/jhipstersampleapplication:latest"
                sh "docker push pipelineregistry.azurecr.io/jhipstersampleapplication:latest"
            }
        }
        stage('Deploy to production') {
            when {
                branch "master"
            }
            steps {
                input message: 'Deploy to production?', ok: 'Deploy!'
                withEnv(["ENV_NAME=prod", "IMAGE_TAG=latest"]) {
                    acsDeploy azureCredentialsId: 'acs-staging',
                        configFilePaths: 'acsK8s.yml',
                        containerRegistryCredentials: [[credentialsId: 'acr',
                                                        url: 'https://pipelineregistry.azurecr.io']],
                        containerService: 'abayerPipelineDemo | Kubernetes',
                        enableConfigSubstitution: true,
                        resourceGroupName: 'abayer-pipeline-demo-acs',
                        sshCredentialsId: 'acs-ssh-creds'
                }
            }
        }
    }
}
