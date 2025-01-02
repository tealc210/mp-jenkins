pipeline {

    environment {
        IMAGE_NAME = "paymybuddy"
        IMAGE_TAG = "latest"
        SONAR_TOKEN = credentials('sonarcloud')
        DOCKERHUB_CREDENTIALS = credentials('DOCKERHUB')
        ENV_PRD = "eazy-prd.agbo.fr"
        ENV_STG = "eazy-stg.agbo.fr"
        ENV_TST = "172.17.0.1"
        DB_HOST_STG = "172.31.28.19"
        DB_HOST_PRD = "172.31.80.69"
    }

    agent none

    stages{

        stage('Scan') {
            agent any
            steps{
                withCredentials([string(credentialsId: 'sonarcloud', variable: 'SONAR_TOKEN')]) {
                    sh 'docker run --rm --name scanner-$BRANCH_NAME -e SONAR_HOST_URL="https://sonarcloud.io" -e SONAR_TOKEN=$SONAR_TOKEN -e SONAR_SCANNER_OPTS="-Dsonar.organization=tealc-210 -Dsonar.projectKey=tealc-210_jenkins" -v "$PWD/app_code/src:/usr/src"  sonarsource/sonar-scanner-cli'
                }
            }
        }

        stage('Init Database') {
            agent any
            steps{
                dir('./app_code/src/main/resources/database/'){
                    sh '''
                    docker ps -a | grep $IMAGE_NAME-$BRANCH_NAME && docker rm -f $IMAGE_NAME-$BRANCH_NAME || echo 'app does not exist'
                    docker ps -a | grep mysql-$BRANCH_NAME && docker rm -f -v mysql-$BRANCH_NAME && docker volume rm sql-$BRANCH_NAME
                    docker container create --name dummy-$BRANCH_NAME -v sql-$BRANCH_NAME:/root hello-world
                    docker cp create.sql dummy-$BRANCH_NAME:/root/create.sql
                    docker rm dummy-$BRANCH_NAME
                    docker run --name mysql-$BRANCH_NAME -p 3306:3306 -v sql-$BRANCH_NAME:/docker-entrypoint-initdb.d -e MYSQL_USER=admin -e MYSQL_PASSWORD=pass -e MYSQL_DATABASE=db_paymybuddy -e MYSQL_ROOT_PASSWORD=password -d mysql:8.0.40-debian
                    sleep 5
                    '''
                }
            }
            
        }

        stage('Build app') {
            agent any
            steps{
                sh 'docker run --rm --name maven-$BRANCH_NAME -v jenkins_jenkins_home:/mnt -w /mnt/workspace/MP_Jenkins_$BRANCH_NAME/app_code/ maven:3-openjdk-17 mvn clean install'
            }
        }

        stage('Build app image') {
            agent any
            steps{
                dir('./app_code/'){
                    script{
                        dockerImage = docker.build("$DOCKERHUB_CREDENTIALS_USR/$IMAGE_NAME-$BRANCH_NAME:$IMAGE_TAG")
                    }
                }
            }
        }

        stage('Run generated image in container') {
            agent any
            steps{
                sh '''
                docker run -d -p 80:8080 --name $IMAGE_NAME-$BRANCH_NAME $IMAGE_NAME-$BRANCH_NAME:$IMAGE_TAG
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
                docker stop $IMAGE_NAME mysql-$BRANCH_NAME
                docker rm -v $IMAGE_NAME mysql-$BRANCH_NAME
                docker volume rm sql-$BRANCH_NAME
                '''
            }
        }

        stage ('Push generated image on docker hub') {
            agent any
            steps {
                script {
                    sh '''
                        docker login -u $DOCKERHUB_CREDENTIALS_USR -p $DOCKERHUB_CREDENTIALS_PSW
                        docker push $DOCKERHUB_CREDENTIALS_USR/$IMAGE_NAME-$BRANCH_NAME:$IMAGE_TAG
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
                        command2="docker pull $DOCKERHUB_CREDENTIALS_USR/$IMAGE_NAME-$BRANCH_NAME:$IMAGE_TAG"
                        command3="docker ps -a | grep $IMAGE_NAME-$BRANCH_NAME && docker rm -f $IMAGE_NAME-$BRANCH_NAME || echo 'app does not exist'"
                        command4="docker run -d -p 80:8080 -e SPRING_DATASOURCE_USERNAME='${DB_CREDS_USR}' -e SPRING_DATASOURCE_PASSWORD='${DB_CREDS_PSW}' -e SPRING_DATASOURCE_URL='jdbc:mysql://${DB_HOST}:3306/db_paymybuddy' --name $IMAGE_NAME-$BRANCH_NAME $DOCKERHUB_CREDENTIALS_USR/$IMAGE_NAME-$BRANCH_NAME:$IMAGE_TAG"
                        ssh -t ubuntu@${DEPLOY_ENV} \
                            -o SendEnv=IMAGE_NAME \
                            -o SendEnv=BRANCH_NAME \
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
                        command2="docker pull $DOCKERHUB_CREDENTIALS_USR/$IMAGE_NAME-$BRANCH_NAME:$IMAGE_TAG"
                        command3="docker ps -a | grep $IMAGE_NAME-$BRANCH_NAME && docker rm -f $IMAGE_NAME-$BRANCH_NAME || echo 'app does not exist'"
                        command4="docker run -d -p 80:8080 -e SPRING_DATASOURCE_USERNAME='${DB_CREDS_USR}' -e SPRING_DATASOURCE_PASSWORD='${DB_CREDS_PSW}' -e SPRING_DATASOURCE_URL='jdbc:mysql://${DB_HOST}:3306/db_paymybuddy' --name $IMAGE_NAME-$BRANCH_NAME $DOCKERHUB_CREDENTIALS_USR/$IMAGE_NAME-$BRANCH_NAME:$IMAGE_TAG"
                        ssh -t ubuntu@${DEPLOY_ENV} \
                            -o SendEnv=IMAGE_NAME \
                            -o SendEnv=BRANCH_NAME \
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
