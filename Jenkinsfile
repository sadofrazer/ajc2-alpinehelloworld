pipeline {

    environment {
        IMAGE_NAME = "alpinehelloworld"
        IMAGE_TAG = "ajc-2.1"
        STAGING = "frazer-ajc-staging-env"
        PRODUCTION = "frazer-ajc-prod-env"
        USERNAME = "sadofrazer"
        CONTAINER_NAME = "alpinehelloworld"
        EC2_PRODUCTION_HOST = "3.230.157.117"
        DEPLOY_APPS = "yes"
    }

    agent any

    stages{

       stage ('Build Image'){
           steps {
               script{
                   sh 'docker build -t $USERNAME/$IMAGE_NAME:$IMAGE_TAG .'
               }
           }
       }

       stage ('clean env and save artifact') {
           environment{
               PASSWORD = credentials('dockerhub_password')
           }
           when {
                expression { GIT_BRANCH == 'origin/master' && DEPLOY_APP != 'yes' }
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

       stage('Push image in staging and deploy it') {
            when {
                expression { GIT_BRANCH == 'origin/master' && DEPLOY_APP != 'yes' }
            }
            agent any
            environment {
                HEROKU_API_KEY = credentials('heroku_api_key')
            }  
            steps {
                script {
                   sh '''
                       heroku container:login
                       heroku create $STAGING || echo "project already exist"
                       heroku container:push -a $STAGING web
                       heroku container:release -a $STAGING web
                    '''
                }
            }
        }

        stage('Push image in Prod and deploy it') {
            when {
                expression { GIT_BRANCH == 'origin/master' && DEPLOY_APP != 'yes' }
            }
            agent any
            environment {
                HEROKU_API_KEY = credentials('heroku_api_key')
            }  
            steps {
                script {
                   sh '''
                       heroku container:login
                       heroku create $PRODUCTION || echo "project already exist"
                       heroku container:push -a $PRODUCTION web
                       heroku container:release -a $PRODUCTION web
                    '''
                }
            }
        }
        
        stage('Deploy app on EC2-cloud Production') {
            agent any
            when{
               expression{ GIT_BRANCH == 'origin/master' && DEPLOY_APP != 'yes'}
            }
            steps{
                withCredentials([sshUserPrivateKey(credentialsId: "ec2_prod_private_key", keyFileVariable: 'keyfile', usernameVariable: 'NUSER')]) {
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                        script{ 

                            timeout(time: 15, unit: "MINUTES") {
                                input message: 'Do you want to approve the deploy in production?', ok: 'Yes'
                            }

                            sh'''
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST} docker run --name $CONTAINER_NAME -d -e PORT=5000 -p 5000:5000 $USERNAME/$IMAGE_NAME:$IMAGE_TAG
                            '''
                        }
                    }
                }
            }
        }
        
         stage('Deploy app on EC2-cloud Production test') {
            agent {
                docker {
                    image('alpine')
                    args ' -u root'
                }
            }
            when{
                expression{ GIT_BRANCH == 'origin/master' && DEPLOY_APP != 'yes'}
            }
            steps{
                withCredentials([sshUserPrivateKey(credentialsId: "ec2_prod_private_key", keyFileVariable: 'keyfile', usernameVariable: 'NUSER')]) {
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                        script{
                            timeout(time: 15, unit: "MINUTES") {
                                input message: 'Do you want to approve the deploy in production?', ok: 'Yes'
                            }						
                            sh'''
                                apk update
                                which ssh-agent || ( apk add openssh-client )
                                eval $(ssh-agent -s)
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST} docker stop $CONTAINER_NAME || true
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST} docker rm $CONTAINER_NAME || true
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST} docker rmi $USERNAME/$IMAGE_NAME:$IMAGE_TAG || true
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST} docker run --name $CONTAINER_NAME -d -e PORT=5000 -p 5000:5000 $USERNAME/$IMAGE_NAME:$IMAGE_TAG
                            '''
                        }
                    }
                }
            }
        }

    }
    
    post {
        success{
            slackSend (color: '#00FF00', message: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
        }
        failure {
            slackSend (color: '#FF0000', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
        }
    }

}
