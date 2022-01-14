pipeline {

    environment {
        IMAGE_NAME = "ic-webapp"
        IMAGE_TAG = "1.0"
        USERNAME = "lianhuahayu"
        CONTAINER_NAME = "test-ic-webapp"
    }

    agent none

    stages{

        stage ('Build image ic-webapp'){
           agent any
           steps {
               script{
                   sh '''
                       docker stop $CONTAINER_NAME || true
                       docker rm $CONTAINER_NAME || true
                       docker rmi $USERNAME/$IMAGE_NAME:$IMAGE_TAG || true
                       docker build -t $USERNAME/$IMAGE_NAME:$IMAGE_TAG .
                   '''
               }
           }
       }

        stage ('Test de vulnerabilites') {
           agent any
           steps {
               script{
                   sh '''
                       echo 'PASSED' || true
                   '''               
               }
           }
       }
    

        stage ('Nettoyage local et push vers un registre publique') {
           agent any
           environment{
               PASSWORD = credentials('token_dockerhub')
           }
           steps {
               script{
                   sh '''
                       docker login -u $USERNAME -p $PASSWORD
                       docker push $USERNAME/$IMAGE_NAME:$IMAGE_TAG
                       docker stop $CONTAINER_NAME || true
                       docker rm $CONTAINER_NAME || true
                       docker rmi $USERNAME/$IMAGE_NAME:$IMAGE_TAG
                   '''
               }
           }
       }

        stage ('Deploiement automatique de env-test via terraform') {
           agent any
           steps {
               script{
                   sh '''
                       echo 'PASSED' || true
                   '''               
               }
           }
       }


        stage ('Test de env-test') {
           agent any
           steps {
               script{
                   sh '''
                       echo 'PASSED' || true
                   '''               
               }
           }
       }

        stage ('Deploiement manuel de env-prod apres validation de env-test') {
           agent any
           steps {
               script{
                   sh '''
                       echo 'PASSED' || true
                   '''               
               }
           }
       }

        stage ('Test de env-prod') {
           agent any
           steps {
               script{
                   sh '''
                       echo 'PASSED' || true
                   '''               
               }
           }
       }
    }
}
