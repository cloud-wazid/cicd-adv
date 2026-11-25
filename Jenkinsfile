
pipeline {
    agent any

    parameters {
        string(
            name: 'APP_NAME',
            defaultValue: 'student-portal',
            description: 'Enter application name'
        )

        choice(
            name: 'DEPLOY_ENV',
            choices: ['dev', 'testing', 'production'],
            description: 'Select deployment environment'
        )
    }

    environment {
        APP_VERSION = '1.0'
        BUILD_FOLDER = 'build'
        DEPLOY_FOLDER = 'deployment'
        DOCKERHUB_REPO = 'wazid2003/student-portal'
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out source code from GitHub...'
                checkout scm
                bat 'dir'
            }
        }

        stage('Initialize') {
            steps {
                echo 'Starting CI/CD Pipeline'

                echo "Application: ${params.APP_NAME}"
                echo "Environment: ${params.DEPLOY_ENV}"
                echo "Build Number: ${BUILD_NUMBER}"
                echo "Version: ${APP_VERSION}"

                bat 'if not exist "%BUILD_FOLDER%" mkdir "%BUILD_FOLDER%"'
            }
        }

        stage('Build Application') {
            steps {
                echo 'Preparing Student Portal application...'

                bat '''
                    if not exist "application\\index.html" (
                        echo HTML application not found!
                        exit /b 1
                    )

                    copy /Y "application\\index.html" "%BUILD_FOLDER%\\index.html"

                    echo Application build completed!
                '''
            }
        }

        stage('Automated Testing') {
            parallel {

                stage('File Test') {
                    steps {
                        echo 'Checking application file...'

                        bat '''
                            if not exist "%BUILD_FOLDER%\\index.html" (
                                echo File test failed!
                                exit /b 1
                            )

                            echo File test passed!
                        '''
                    }
                }

                stage('Content Test') {
                    steps {
                        echo 'Checking application content...'

                        bat '''
                            findstr /C:"Welcome to Student Portal" "%BUILD_FOLDER%\\index.html"

                            if errorlevel 1 exit /b 1

                            echo Content test passed!
                        '''
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                echo 'Building Docker image...'

                bat '''
                    docker build ^
                    -t "%DOCKERHUB_REPO%:%BUILD_NUMBER%" ^
                    -t "%DOCKERHUB_REPO%:latest" .
                '''
            }
        }

        stage('Docker Push') {
            steps {
                echo 'Pushing Docker image to Docker Hub...'

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKERHUB_USERNAME',
                        passwordVariable: 'DOCKERHUB_PASSWORD'
                    )
                ]) {

                    bat 'powershell -NoProfile -NonInteractive -Command "$env:DOCKERHUB_PASSWORD | docker login --username $env:DOCKERHUB_USERNAME --password-stdin"'

                    bat '''
                        docker push "%DOCKERHUB_REPO%:%BUILD_NUMBER%"
                        if errorlevel 1 exit /b 1

                        docker push "%DOCKERHUB_REPO%:latest"
                        if errorlevel 1 exit /b 1

                        docker logout
                    '''
                }
            }
        }

        stage('Manual Approval') {
            when {
                expression {
                    params.DEPLOY_ENV == 'production'
                }
            }

            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    input(
                        message: 'Approve production deployment?',
                        ok: 'Deploy Now'
                    )
                }
            }
        }

        stage('Deploy Application') {
            steps {
                script {
                    try {
                        echo "Deploying ${params.APP_NAME} to ${params.DEPLOY_ENV}"

                        bat '''
                            docker rm -f student-portal 2>NUL

                            docker run -d ^
                            --restart unless-stopped ^
                            --name student-portal ^
                            -p 8085:80 ^
                            "%DOCKERHUB_REPO%:%BUILD_NUMBER%"
                        '''

                        echo 'Deployment completed successfully!'

                    } catch (Exception e) {
                        echo "Deployment error: ${e.getMessage()}"
                        throw e
                    }
                }
            }
        }

        stage('Archive Artifacts') {
            steps {
                archiveArtifacts(
                    artifacts: 'build/index.html',
                    fingerprint: true
                )

                echo 'Application artifact archived!'
            }
        }
    }

    post {
        success {
            echo 'CI/CD Pipeline completed successfully!'
            echo "Application: ${params.APP_NAME}"
            echo "Environment: ${params.DEPLOY_ENV}"
        }

        failure {
            echo 'Pipeline failed. Check console output.'
        }

        aborted {
            echo 'Pipeline was aborted.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}
