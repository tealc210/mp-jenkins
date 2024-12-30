@Library('remote_deploy')_

pipeline {
    parameters{
        string defaultValue: 'Jenkinsfile', description: '', name: 'JenkinsFileParameter', trim: false
    }
    environment {
        IMAGE_NAME = "paymybuddy"
        IMAGE_TAG = "latest"
        SONAR_TOKEN = credentials('sonarcloud')
        DOCKERHUB_CREDENTIALS = credentials('DOCKERHUB')
    }

    agent none

    stages{
        stage('Scan') {
            agent any
            steps{
                withCredentials([string(credentialsId: 'sonarcloud', variable: 'SONAR_TOKEN')]) {
                    sh 'docker run --rm -e SONAR_HOST_URL="https://sonarcloud.io" -e SONAR_TOKEN=$SONAR_TOKEN -e SONAR_SCANNER_OPTS="-Dsonar.organization=tealc-210 -Dsonar.projectKey=tealc-210_jenkins" -v "$PWD/app_code/src:/usr/src"  sonarsource/sonar-scanner-cli'
                }
            }
        }

        stage('Init Database') {
            agent any
            steps{
                dir('./app_code/src/main/resources/database/'){
                    sh '''
                    docker ps -a | grep $IMAGE_NAME && docker rm -f $IMAGE_NAME || echo 'app does not exist'
                    docker ps -a | grep mysql && docker stop mysql && docker rm -v mysql && docker volume rm sql
                    docker container create --name dummy -v sql:/root hello-world
                    docker cp create.sql dummy:/root/create.sql
                    docker rm dummy
                    docker run --name mysql -p 3306:3306 -v sql:/docker-entrypoint-initdb.d -e MYSQL_USER=admin -e MYSQL_PASSWORD=pass -e MYSQL_DATABASE=db_paymybuddy -e MYSQL_ROOT_PASSWORD=password -d mysql:8.0.40-debian
                    sleep 5
                    '''
                }
            }
            
        }

        stage('Build app') {
            agent any
            steps{
                withSonarQubeEnv('SonarCloud') {
                    sh 'docker run --rm --name maven -v jenkins_jenkins_home:/mnt -w /mnt/workspace/mp-jenkins/app_code/ maven:3-openjdk-17 mvn clean install'
                }
            }
        }
        stage('Build app image') {
            agent any
            steps{
                dir('./app_code/'){
                    script{
                        dockerImage = docker.build("tealc210/$IMAGE_NAME:$IMAGE_TAG")
                    }
                }
            }
        }
        stage('Run generated image in container') {
            agent any
            steps{
                sh '''
                docker run -d -p 80:8080 --name $IMAGE_NAME $IMAGE_NAME:$IMAGE_TAG
                sleep 30
                '''
            }
        }
        stage('Check application') {
            agent any
            steps{
                sh 'curl -L http://$ENV_TST | grep "Pay My Buddy button"'
            }
        }
        stage('Cleanup') {
            agent any
            steps{
                sh '''
                docker stop $IMAGE_NAME mysql
                docker rm -v $IMAGE_NAME mysql
                docker volume rm sql
                '''
            }
        }

        stage ('Push generated image on docker hub') {
            agent any
            steps {
                script {
                    sh '''
                        docker login -u $DOCKERHUB_CREDENTIALS_USR -p $DOCKERHUB_CREDENTIALS_PSW
                        docker push $DOCKERHUB_CREDENTIALS_USR/$IMAGE_NAME:$IMAGE_TAG
                    '''
                }
            }
        }

        stage ('Deploy to Staging Env') {
            agent any
            when {
                not {
                    branch 'main'
                    }
            }
            environment {
                DEPLOY_ENV = "${ENV_STG}"
                DB_HOST = "${DB_HOST_STG}"
                DB_CREDS = credentials('DB_CREDS')
            }
            steps {
                sshagent(credentials: ['SSHKEY']) {
                    sh '''
                        [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                        ssh-keyscan -t rsa,dsa,ed25519 ${DEPLOY_ENV} >> ~/.ssh/known_hosts
                        command1="docker login -u $DOCKERHUB_CREDENTIALS_USR -p $DOCKERHUB_CREDENTIALS_PSW"
                        command2="docker pull $DOCKERHUB_CREDENTIALS_USR/$IMAGE_NAME:$IMAGE_TAG"
                        command3="docker ps -a | grep $IMAGE_NAME && docker rm -f $IMAGE_NAME || echo 'app does not exist'"
                        command4="docker run -d -p 80:8080 -e SPRING_DATASOURCE_USERNAME='${DB_CREDS_USR}' -e SPRING_DATASOURCE_PASSWORD='${DB_CREDS_PSW}' -e SPRING_DATASOURCE_URL='jdbc:mysql://${DB_HOST}:3306/db_paymybuddy' --name $IMAGE_NAME $DOCKERHUB_CREDENTIALS_USR/$IMAGE_NAME:$IMAGE_TAG"
                        ssh -t ubuntu@${DEPLOY_ENV} \
                            -o SendEnv=IMAGE_NAME \
                            -o SendEnv=IMAGE_TAG \
                            -o SendEnv=DOCKERHUB_CREDENTIALS_USR \
                            -o SendEnv=DOCKERHUB_CREDENTIALS_PSW \
                            -C "$command1 && $command2 && $command3 && $command4 && sleep 30"
                    '''
                }
            }
        }
        stage('Check staging deployed application') {
            agent any
            when {
                not {
                    branch 'main'
                    }
            }
            steps{
                sh 'curl -L http://$ENV_STG | grep "Pay My Buddy button"'
            }
        }

        stage ('Deploy to Prod Env') {
            agent any
            when {
                branch 'main'
            }
            environment {
                DEPLOY_ENV = "${ENV_PRD}"
                DB_HOST = "${DB_HOST_PRD}"
                DB_CREDS = credentials('DB_CREDS')
            }
            steps {
                sshagent(credentials: ['SSHKEY']) {
                    sh '''
                        [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                        ssh-keyscan -t rsa,dsa,ed25519 ${DEPLOY_ENV} >> ~/.ssh/known_hosts
                        command1="docker login -u $DOCKERHUB_CREDENTIALS_USR -p $DOCKERHUB_CREDENTIALS_PSW"
                        command2="docker pull $DOCKERHUB_CREDENTIALS_USR/$IMAGE_NAME:$IMAGE_TAG"
                        command3="docker ps -a | grep $IMAGE_NAME && docker rm -f $IMAGE_NAME || echo 'app does not exist'"
                        command4="docker run -d -p 80:8080 -e SPRING_DATASOURCE_USERNAME='${DB_CREDS_USR}' -e SPRING_DATASOURCE_PASSWORD='${DB_CREDS_PSW}' -e SPRING_DATASOURCE_URL='jdbc:mysql://${DB_HOST}:3306/db_paymybuddy' --name $IMAGE_NAME $DOCKERHUB_CREDENTIALS_USR/$IMAGE_NAME:$IMAGE_TAG"
                        ssh -t ubuntu@${DEPLOY_ENV} \
                            -o SendEnv=IMAGE_NAME \
                            -o SendEnv=IMAGE_TAG \
                            -o SendEnv=DOCKERHUB_CREDENTIALS_USR \
                            -o SendEnv=DOCKERHUB_CREDENTIALS_PSW \
                            -C "$command1 && $command2 && $command3 && $command4 && sleep 30"
                    '''
                }
            }
        }
        stage('Check Prod deployed application') {
            agent any
            when {
                branch 'main'
            }
            steps{
                sh 'curl -L http://$ENV_PRD | grep "Pay My Buddy button"'
            }
        }
    }
}
