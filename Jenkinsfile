pipeline {
    agent any
    
    stages {
        stage('Cleanup Everything') {
            steps {
                sh '''
                    echo "=== Cleaning up old containers and networks ==="
                    docker stop backend1 backend2 nginx-lb 2>/dev/null || true
                    docker rm backend1 backend2 nginx-lb 2>/dev/null || true
                    docker network rm lab6-network 2>/dev/null || true
                '''
            }
        }
        
        stage('Create Network') {
            steps {
                sh '''
                    echo "=== Creating dedicated network ==="
                    docker network create lab6-network
                    echo "Network created successfully"
                '''
            }
        }
        
        stage('Build Backend Image') {
            steps {
                sh '''
                    echo "=== Building backend image ==="
                    cd backend
                    docker build -t backend-app .
                    docker images | grep backend-app
                '''
            }
        }
        
        stage('Deploy Backends') {
            steps {
                sh '''
                    echo "=== Deploying backend containers on lab6-network ==="
                    
                    # Run backend1
                    docker run -d --name backend1 \
                      --network lab6-network \
                      backend-app
                    
                    # Run backend2
                    docker run -d --name backend2 \
                      --network lab6-network \
                      backend-app
                    
                    echo "Waiting for backends to initialize..."
                    sleep 5
                    
                    # Verify backends are running
                    echo "=== Running Containers ==="
                    docker ps | grep backend
                    
                    # Check backend logs
                    echo "=== Backend1 Logs ==="
                    docker logs backend1
                    
                    echo "=== Backend2 Logs ==="
                    docker logs backend2
                '''
            }
        }
        
        stage('Verify Backend Connectivity') {
            steps {
                sh '''
                    echo "=== Verifying backends are reachable ==="
                    
                    # Install curl in backends if needed (for testing)
                    docker exec backend1 apt-get update && docker exec backend1 apt-get install -y curl || true
                    
                    # Test if backends are listening on port 8080
                    echo "Testing backend1:8080 from inside backend1:"
                    docker exec backend1 curl -I http://localhost:8080 || echo "Backend1 not responding locally"
                    
                    echo "Testing backend2:8080 from inside backend2:"
                    docker exec backend2 curl -I http://localhost:8080 || echo "Backend2 not responding locally"
                '''
            }
        }
        
        stage('Deploy Nginx') {
            steps {
                sh '''
                    echo "=== Deploying Nginx on lab6-network ==="
                    
                    # Create nginx config with proper upstream
                    cat > nginx/default.conf << 'EOF'
upstream backend_servers {
    server backend1:8080;
    server backend2:8080;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://backend_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
EOF
                    
                    echo "=== Nginx Config Created ==="
                    cat nginx/default.conf
                    
                    # Run nginx container on the same network
                    docker run -d --name nginx-lb \
                      --network lab6-network \
                      -p 80:80 \
                      nginx:latest
                    
                    echo "Waiting for nginx to start..."
                    sleep 5
                    
                    # Copy nginx configuration
                    docker cp nginx/default.conf nginx-lb:/etc/nginx/conf.d/default.conf
                    
                    # Test nginx configuration
                    echo "=== Testing Nginx Configuration ==="
                    docker exec nginx-lb nginx -t
                    
                    # Reload nginx
                    docker exec nginx-lb nginx -s reload
                    
                    echo "Nginx configured and reloaded successfully"
                '''
            }
        }
        
        stage('Test Network Connectivity') {
            steps {
                sh '''
                    echo "=== Testing Network Connectivity ==="
                    
                    # Show network details
                    echo "Network Details:"
                    docker network inspect lab6-network | grep -A 10 "Containers"
                    
                    # Test if nginx can ping backends
                    echo "Ping test from nginx to backend1:"
                    docker exec nginx-lb ping -c 2 backend1 || echo "❌ Cannot ping backend1"
                    
                    echo "Ping test from nginx to backend2:"
                    docker exec nginx-lb ping -c 2 backend2 || echo "❌ Cannot ping backend2"
                    
                    # Test if nginx can access backends via HTTP
                    echo "HTTP test from nginx to backend1:8080:"
                    docker exec nginx-lb apt-get update && docker exec nginx-lb apt-get install -y curl || true
                    docker exec nginx-lb curl -I http://backend1:8080 || echo "❌ Cannot access backend1:8080"
                    
                    echo "HTTP test from nginx to backend2:8080:"
                    docker exec nginx-lb curl -I http://backend2:8080 || echo "❌ Cannot access backend2:8080"
                    
                    # Test DNS resolution
                    echo "DNS resolution test:"
                    docker exec nginx-lb getent hosts backend1 || echo "❌ Cannot resolve backend1"
                    docker exec nginx-lb getent hosts backend2 || echo "❌ Cannot resolve backend2"
                '''
            }
        }
        
        stage('Test Load Balancer') {
            steps {
                sh '''
                    echo "=== Testing Load Balancer ==="
                    echo "Making 6 requests to http://localhost:"
                    for i in 1 2 3 4 5 6; do
                        echo -n "Request $i: "
                        curl -s http://localhost | grep -o "Served by.*" || echo "No response or different format"
                        sleep 1
                    done
                '''
            }
        }
    }
    
    post {
        always {
            sh '''
                echo "=== Final Container Status ==="
                docker ps
                
                echo "=== Network Status ==="
                docker network ls | grep lab6-network
            '''
        }
        
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed. Debugging information:'
            sh '''
                echo "=== Debug Information ==="
                echo "1. All containers:"
                docker ps -a
                
                echo "2. Network details:"
                docker network inspect lab6-network || true
                
                echo "3. Backend logs:"
                docker logs backend1 || true
                docker logs backend2 || true
                
                echo "4. Nginx logs:"
                docker logs nginx-lb || true
                
                echo "5. Nginx config:"
                docker exec nginx-lb cat /etc/nginx/conf.d/default.conf || true
            '''
        }
    }
}
