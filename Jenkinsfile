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
                sh '''
                docker run --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=password -d mysql
                sleep 5
                cd app_code/src/main/resources/database
                cat create.sql data.sql | mysql -h $(ip a show docker0 | awk '{print $4}' | cut -d/ -f1) -u root -ppassword
                '''
            }
            
        }
        stage('Build app') {
            agent any
            steps{
                sh 'docker run --rm --name maven -v "/$(pwd)/app_code/":/mnt -w /mnt maven:3-openjdk-17 mvn clean install'
            }
        }
        stage('Build app image') {
            agent any
            
            steps{
                sh 'docker build -t $IMAGE_NAME:$IMAGE_TAG .'
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
                docker rm $IMAGE_NAME mysql
                '''
            }
        }
    }
}
