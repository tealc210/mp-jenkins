pipeline {

    environment {
        IMAGE_NAME = "paymybuddy"
        IMAGE_TAG = "latest"
        ENV_PRD =""
        ENV_STG =""
        ENV_TST = "172.17.0.1"
    }

    agent none

    stages{
        /*stage('Init Database') {
            agent any
            steps{
                dir('./app_code/src/main/resources/database/'){
                    sh '''
                    docker ps -a | grep mysql && docker stop mysql && docker rm -v mysql && docker volume rm sql
                    docker container create --name dummy -v sql:/root hello-world
                    docker cp create.sql dummy:/root/create.sql
                    docker rm dummy
                    docker run --name mysql -p 3306:3306 -v sql:/docker-entrypoint-initdb.d -e MYSQL_USER=admin -e MYSQL_PASSWORD=pass -e MYSQL_DATABASE=db_paymybuddy -e MYSQL_ROOT_PASSWORD=password -d mysql:8.0.40-debian
                    sleep 5
                    '''
                }
            }
            
        }*/
        stage('Scan') {
            agent any
            steps{
                withSonarQubeEnv('SonarCloud') {
                //withCredentials([string(credentialsId: 'sonarcloud', variable: 'SONARCLOUD_TOKEN')]) {
                    sh 'docker run --rm --name maven -v jenkins_jenkins_home:/mnt -w /mnt/workspace/mp-jenkins/app_code/ maven:3-openjdk-17 mvn verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.organization=tealc-210 -Dsonar.projectKey=tealc-210_jenkins -Dsonar.sources=. -Dsonar.host.url=https://sonarcloud.io'
                }
            }
        }
        /*stage('Build app') {
            agent any
            steps{
                withSonarQubeEnv('SonarCloud') {
                    sh 'docker run --rm --name maven -v jenkins_jenkins_home:/mnt -w /mnt/workspace/mp-jenkins/app_code/ maven:3-openjdk-17 mvn clean install'
                }
            }
        }*/
        /*stage("Quality Gate"){
            timeout(time: 1, unit: 'HOURS') {
                def qg = waitForQualityGate()
                if (qg.status != 'OK') {
                    error "Pipeline aborted due to quality gate failure: ${qg.status}"
                }
            }
        }*/
        stage('Build app image') {
            agent any
            
            steps{
                dir('./app_code/'){
                    script{
                        dockerImage = docker.build("$IMAGE_NAME:$IMAGE_TAG")
                    }
                }
                //sh 'docker build -t $IMAGE_NAME:$IMAGE_TAG /var/lib/docker/volumes/jenkins_jenkins_home/_data/workspace/mp-jenkins/app_code/'
            }
        }
        stage('Run generated image in container') {
            agent any
            
            steps{
                sh '''
                docker run -d -p 80:8080 --name $IMAGE_NAME $IMAGE_NAME:$IMAGE_TAG
                sleep 10
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

/*        stage ('Login and Push Image on docker hub') {
            agent any           
            steps {
                script {
                    sh '''
                        docker login -u $DOCKERHUB_AUTH_USR -p $DOCKERHUB_AUTH_PSW
                        docker push ${ID_DOCKER}/$IMAGE_NAME:$IMAGE_TAG
                    '''
                }
            }
        }

        stage ('Deploy in staging') {
            agent any
            environment {
                HOSTNAME_DEPLOY_STAGING = "ec2-100-24-13-223.compute-1.amazonaws.com"
            }
            steps {
                sshagent(credentials: ['SSH_AUTH_SERVER']) {
                    sh '''
                        [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                        ssh-keyscan -t rsa,dsa ${HOSTNAME_DEPLOY_STAGING} >> ~/.ssh/known_hosts
                        command1="docker login -u $DOCKERHUB_AUTH_USR -p $DOCKERHUB_AUTH_PSW"
                        command2="docker pull $DOCKERHUB_AUTH_USR/$IMAGE_NAME:$IMAGE_TAG"
                        command3="docker rm -f webapp || echo 'app does not exist'"
                        command4="docker run -d -p 80:5000 -e PORT=5000 --name webapp $DOCKERHUB_AUTH_USR/$IMAGE_NAME:$IMAGE_TAG"
                        ssh -t centos@${HOSTNAME_DEPLOY_STAGING} \
                            -o SendEnv=IMAGE_NAME \
                            -o SendEnv=IMAGE_TAG \
                            -o SendEnv=DOCKERHUB_AUTH_USR \
                            -o SendEnv=DOCKERHUB_AUTH_PSW \
                            -C "$command1 && $command2 && $command3 && $command4"
                    '''
                }
            }
        }

        stage ('Deploy in prod') {
            agent any
            environment {
                HOSTNAME_DEPLOY_PROD = "ec2-50-17-119-91.compute-1.amazonaws.com"
            }
            steps {
                sshagent(credentials: ['SSH_AUTH_PROD']) {
                    sh '''
                        [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                        ssh-keyscan -t rsa,dsa ${HOSTNAME_DEPLOY_PROD} >> ~/.ssh/known_hosts
                        command1="docker login -u $DOCKERHUB_AUTH_USR -p $DOCKERHUB_AUTH_PSW"
                        command2="docker pull $DOCKERHUB_AUTH_USR/$IMAGE_NAME:$IMAGE_TAG"
                        command3="docker rm -f webapp || echo 'app does not exist'"
                        command4="docker run -d -p 80:5000 -e PORT=5000 --name webapp $DOCKERHUB_AUTH_USR/$IMAGE_NAME:$IMAGE_TAG"
                        ssh -t centos@${HOSTNAME_DEPLOY_PROD} \
                            -o SendEnv=IMAGE_NAME \
                            -o SendEnv=IMAGE_TAG \
                            -o SendEnv=DOCKERHUB_AUTH_USR \
                            -o SendEnv=DOCKERHUB_AUTH_PSW \
                            -C "$command1 && $command2 && $command3 && $command4"
                    '''
                }
            }
        } */
    }
}
