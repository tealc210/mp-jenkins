pipeline {

    environment {
        IMAGE_NAME = "paymybuddy"
        IMAGE_TAG = "latest"
        ENV_PRD =""
        ENV_STG =""
    }

    agent none

    stages{
        stage('Init Database') {
            agent any
            steps{
                dir('./app_code/src/main/resources/database/'){
                    sh '''
                    docker ps -a | grep mysql && docker stop mysql && docker rm mysql && docker volume rm sql
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
                sh 'docker run --rm --name maven -v jenkins_jenkins_home:/mnt -w /mnt/workspace/mp-jenkins/app_code/ maven:3-openjdk-17 mvn clean install'
            }
        }
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
                sh 'curl http://localhost | grep -i buddy'
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
    }
}
