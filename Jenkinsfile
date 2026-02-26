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
                    echo "If CC_Lab-6 exists, listing its contents:"
                    ls -la CC_Lab-6 || echo "CC_Lab-6 directory not found!"
                '''
            }
        }
        
        stage('Cleanup Old Containers') {
            steps {
                sh '''
                    docker rm -f backend1 backend2 nginx-lb 2>/dev/null || true
                '''
            }
        }
        
        stage('Build Backend Image') {
            steps {
                script {
                    // Check if CC_Lab-6 exists, if not try current directory
                    sh '''
                        if [ -d "CC_Lab-6" ]; then
                            cd CC_Lab-6
                            docker build -t backend-app backend
                        elif [ -d "backend" ]; then
                            docker build -t backend-app backend
                        else
                            echo "ERROR: Cannot find backend directory!"
                            exit 1
                        fi
                    '''
                }
            }
        }
        
        stage('Deploy Backends') {
            steps {
                sh '''
                    docker run -d --name backend1 backend-app
                    docker run -d --name backend2 backend-app
                    sleep 5
                '''
            }
        }
        
        stage('Deploy Nginx') {
            steps {
                sh '''
                    docker run -d --name nginx-lb -p 80:80 nginx:latest
                    sleep 3
                    # Try different possible paths for nginx config
                    if [ -f "CC_Lab-6/nginx/default.conf" ]; then
                        docker cp CC_Lab-6/nginx/default.conf nginx-lb:/etc/nginx/conf.d/default.conf
                    elif [ -f "nginx/default.conf" ]; then
                        docker cp nginx/default.conf nginx-lb:/etc/nginx/conf.d/default.conf
                    else
                        echo "WARNING: Could not find nginx config, creating default one"
                        echo 'upstream backend_servers { server backend1:8080; server backend2:8080; } server { listen 80; location / { proxy_pass http://backend_servers; } }' | docker exec -i nginx-lb tee /etc/nginx/conf.d/default.conf
                    fi
                    docker exec nginx-lb nginx -s reload
                '''
            }
        }
        
        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "=== Running Containers ==="
                    docker ps
                    echo "=== Testing Backend1 ==="
                    docker exec backend1 wget -q -O- http://localhost:8080 || echo "Backend1 not responding"
                    echo "=== Testing Nginx ==="
                    curl -I http://localhost || echo "Nginx not responding"
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
        }
    }
}
