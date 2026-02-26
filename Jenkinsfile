pipeline {
    agent any
    
    stages {
        stage('Cleanup Old Containers') {
            steps {
                sh '''
                    docker rm -f backend1 backend2 nginx-lb 2>/dev/null || true
                '''
            }
        }
        
        stage('Build Backend Image') {
            steps {
                sh '''
                    cd CC_Lab-6
                    docker build -t backend-app backend
                '''
            }
        }
        
        stage('Deploy Backends') {
            steps {
                sh '''
                    cd CC_Lab-6
                    docker run -d --name backend1 backend-app
                    docker run -d --name backend2 backend-app
                    sleep 5
                '''
            }
        }
        
        stage('Deploy Nginx') {
            steps {
                sh '''
                    cd CC_Lab-6
                    docker run -d --name nginx-lb -p 80:80 nginx:latest
                    sleep 3
                    docker cp nginx/default.conf nginx-lb:/etc/nginx/conf.d/default.conf
                    docker exec nginx-lb nginx -s reload
                '''
            }
        }
        
        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "Checking running containers:"
                    docker ps
                    echo "Testing backend1:"
                    docker exec backend1 wget -q -O- http://localhost:8080 || true
                    echo "Testing nginx:"
                    curl -I http://localhost || true
                '''
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the logs above.'
        }
    }
}
