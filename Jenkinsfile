pipeline {
    agent any
    
    stages {
        stage('Debug - List Files') {
            steps {
                sh '''
                    echo "Current directory:"
                    pwd
                    echo "Files in workspace:"
                    ls -la
                '''
            }
        }
        
        stage('Cleanup Old Containers') {
            steps {
                sh '''
                    docker rm -f backend1 backend2 nginx-lb 2>/dev/null || true
                    docker network rm lab6-network 2>/dev/null || true
                '''
            }
        }
        
        stage('Create Network') {
            steps {
                sh '''
                    docker network create lab6-network
                '''
            }
        }
        
        stage('Build Backend Image') {
            steps {
                sh '''
                    docker build -t backend-app backend
                '''
            }
        }
        
        stage('Deploy Backends') {
            steps {
                sh '''
                    docker run -d --name backend1 --network lab6-network backend-app
                    docker run -d --name backend2 --network lab6-network backend-app
                    sleep 5
                '''
            }
        }
        
        stage('Deploy Nginx') {
            steps {
                sh '''
                    docker run -d --name nginx-lb -p 80:80 --network lab6-network nginx:latest
                    sleep 3
                    docker cp nginx/default.conf nginx-lb:/etc/nginx/conf.d/default.conf
                    docker exec nginx-lb nginx -s reload
                    sleep 2
                '''
            }
        }
        
        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "=== Running Containers ==="
                    docker ps
                    echo "=== Testing Backend1 directly ==="
                    docker exec backend1 wget -q -O- http://localhost:8080 || echo "Backend1 not responding"
                    echo "=== Testing Backend2 directly ==="
                    docker exec backend2 wget -q -O- http://localhost:8080 || echo "Backend2 not responding"
                    echo "=== Testing Nginx Load Balancer ==="
                    curl -I http://localhost || echo "Nginx not responding"
                    echo "=== Testing Load Balancing (should alternate) ==="
                    for i in 1 2 3 4; do
                        echo "Request $i:"
                        curl -s http://localhost | grep "Served by" || echo "No response"
                        sleep 1
                    done
                '''
            }
        }
    }
    
    post {
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed. Check the logs above.'
            sh '''
                echo "=== Debug Information ==="
                echo "1. Running containers:"
                docker ps -a
                echo "2. Network details:"
                docker network inspect lab6-network || true
                echo "3. Backend logs:"
                docker logs backend1 || true
                docker logs backend2 || true
                echo "4. Nginx config:"
                docker exec nginx-lb cat /etc/nginx/conf.d/default.conf || true
            '''
        }
    }
}
