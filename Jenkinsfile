pipeline {
    agent {
        label "docker"
    }
    environment {
        REL_VERSION = "${BRANCH_NAME.contains('release-') ? BRANCH_NAME.drop(BRANCH_NAME.lastIndexOf('-')+1) + '.' + BUILD_NUMBER : 'M.' + BUILD_NUMBER}"
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
                }
            }
        }
        stage('Build Container') {
            environment {
                DOCKERHUB = credentials("dockerhub")
            }
            steps {
                sh "docker login -u '${DOCKERHUB_USR}' -p '${DOCKERHUB_PSW}'"
                sh "./mvnw -B docker:build -Ddocker-tag=${BUILD_ID}"
                sh "docker push abayer1138/jhipstersampleapplication:${BUILD_ID}"
            }
        }
        stage('Deploy to Staging') {
            steps {
                echo "Let's pretend a deployment is happening"
            }
        }
        stage('Deploy to production') {
            steps {
                input message: 'Deploy to production?', ok: 'Fire zee missiles!'
                echo "Let's pretend a production deployment is happening"
            }
        }
    }
}
