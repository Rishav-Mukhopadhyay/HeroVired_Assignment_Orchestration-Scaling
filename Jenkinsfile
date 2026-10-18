// Posts a message to the SNS topic -> Slack (via Amazon Q Developer) + email.
// The JSON uses the "custom notification" format that Amazon Q Developer understands.
def notifySns(String status) {
    sh """
        aws sns publish --region ${env.AWS_REGION} --topic-arn ${env.SNS_TOPIC_ARN} \
          --message '{"version":"1.0","source":"custom","content":{"textType":"client-markdown","title":"${status}: ${env.JOB_NAME} #${env.BUILD_NUMBER}","description":"Image tag: ${env.IMAGE_TAG}\\nDetails: ${env.BUILD_URL}"}}'
    """
}

pipeline {
    agent any    // run on the Jenkins server itself

    // "Sticky notes" (variables) available to every stage below
    environment {
        AWS_REGION     = 'ap-south-1'
        AWS_ACCOUNT_ID = '722867460821'
        REGISTRY       = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_TAG      = "${BUILD_NUMBER}"   // Jenkins counts builds: 1, 2, 3...
        SNS_TOPIC_ARN  = "arn:aws:sns:${AWS_REGION}:${AWS_ACCOUNT_ID}:streamingapp-alerts"

        // Frontend uses relative URLs; nginx (frontend/nginx.conf) forwards /api/... to the backends
        REACT_APP_AUTH_API_URL         = '/api'
        REACT_APP_STREAMING_API_URL    = '/api'
        REACT_APP_STREAMING_PUBLIC_URL = '/'
        REACT_APP_ADMIN_API_URL        = '/api/admin'
        REACT_APP_CHAT_API_URL         = '/api/chat'
        REACT_APP_CHAT_SOCKET_URL      = '/'

    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm    // "scm" = the Git repo this job is connected to
            }
        }

        stage('Login to ECR') {
            steps {
                sh 'aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $REGISTRY'
            }
        }

        stage('Build images') {
            steps {
                // auth is built from inside its own folder
                sh 'docker build -t $REGISTRY/streamingapp-auth:$IMAGE_TAG backend/authService'

                // these three are built from the backend/ folder (same as docker-compose.yml)
                sh 'docker build -t $REGISTRY/streamingapp-streaming:$IMAGE_TAG -f backend/streamingService/Dockerfile backend'
                sh 'docker build -t $REGISTRY/streamingapp-admin:$IMAGE_TAG -f backend/adminService/Dockerfile backend'
                sh 'docker build -t $REGISTRY/streamingapp-chat:$IMAGE_TAG -f backend/chatService/Dockerfile backend'

                sh '''
                    docker build -t $REGISTRY/streamingapp-frontend:$IMAGE_TAG \
                      --build-arg REACT_APP_AUTH_API_URL=$REACT_APP_AUTH_API_URL \
                      --build-arg REACT_APP_STREAMING_API_URL=$REACT_APP_STREAMING_API_URL \
                      --build-arg REACT_APP_STREAMING_PUBLIC_URL=$REACT_APP_STREAMING_PUBLIC_URL \
                      --build-arg REACT_APP_ADMIN_API_URL=$REACT_APP_ADMIN_API_URL \
                      --build-arg REACT_APP_CHAT_API_URL=$REACT_APP_CHAT_API_URL \
                      --build-arg REACT_APP_CHAT_SOCKET_URL=$REACT_APP_CHAT_SOCKET_URL \
                      frontend
                '''
            }
        }

        stage('Push images') {
            steps {
                sh '''
                    for svc in auth streaming admin chat frontend; do
                      docker push $REGISTRY/streamingapp-$svc:$IMAGE_TAG
                      docker tag  $REGISTRY/streamingapp-$svc:$IMAGE_TAG $REGISTRY/streamingapp-$svc:latest
                      docker push $REGISTRY/streamingapp-$svc:latest
                    done
                '''
            }
        }
    }

    post {
        always {
            sh 'docker image prune -f'   // delete leftover images so the disk doesn't fill up
        }
        success {
            echo "✅ Build ${BUILD_NUMBER}: all 5 images pushed to ECR"
            notifySns('✅ BUILD SUCCEEDED')
        }
        failure {
            echo "❌ Build ${BUILD_NUMBER} failed. Check the stage that turned red."
            notifySns('❌ BUILD FAILED')
        }
    }
}
