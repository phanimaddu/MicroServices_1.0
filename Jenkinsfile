pipeline {

    agent any

    parameters {

        string(
            name: 'IMAGE_TAG',
            defaultValue: '2.0',
            description: 'Docker image version'
        )

        choice(
            name: 'ACTION',
            choices: ['BUILD_AND_DEPLOY', 'BUILD_ONLY', 'DEPLOY_ONLY'],
            description: 'Pipeline action'
        )
    }

    environment {

        AWS_REGION = 'us-east-1'

        AWS_ACCOUNT_ID = '658606186375'

        EC2_HOST = '172.31.5.188'

        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
    }

    stages {

        // ================= CHECKOUT =================

        stage('Checkout') {
            steps {
                git branch: 'feature/docker',
                    credentialsId: 'github-credentials',
                    url: 'https://github.com/phanimaddu/MicroServices_1.0.git'
            }
        }

        // ================= VALIDATE =================

        stage('Validate') {
            steps {
                sh '''
                    node --version
                    npm --version
                    npm ci --omit=dev
                    docker compose config --quiet
                '''
            }
        }

        // ================= BUILD IMAGES =================

        stage('Build Images') {

            when {
                expression {
                    params.ACTION == 'BUILD_AND_DEPLOY' ||
                    params.ACTION == 'BUILD_ONLY'
                }
            }

            steps {
                script {

                    def services = [
                        [name: 'user-service', file: 'Dockerfile.user'],
                        [name: 'product-service', file: 'Dockerfile.product'],
                        [name: 'order-service', file: 'Dockerfile.order']
                    ]

                    services.each { service ->

                        sh """
                            docker build \
                            -f ${service.file} \
                            -t ${ECR_REGISTRY}/${service.name}:${params.IMAGE_TAG} .
                        """
                    }
                }
            }
        }

        // ================= ECR LOGIN =================

        stage('ECR Login') {

            when {
                expression {
                    params.ACTION == 'BUILD_AND_DEPLOY' ||
                    params.ACTION == 'BUILD_ONLY'
                }
            }

            steps {

                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-jenkins'
                ]]) {

                    sh '''
                        aws ecr get-login-password \
                        --region $AWS_REGION \
                        | docker login \
                        --username AWS \
                        --password-stdin $ECR_REGISTRY
                    '''
                }
            }
        }

        // ================= PUSH IMAGES =================

        stage('Push Images') {

            when {
                expression {
                    params.ACTION == 'BUILD_AND_DEPLOY' ||
                    params.ACTION == 'BUILD_ONLY'
                }
            }

            steps {
                sh '''
                    docker push $ECR_REGISTRY/user-service:$IMAGE_TAG
                    docker push $ECR_REGISTRY/product-service:$IMAGE_TAG
                    docker push $ECR_REGISTRY/order-service:$IMAGE_TAG
                '''
            }
        }

        // ================= DEPLOY TO EC2 =================

        stage('Deploy to EC2') {

            when {
                expression {
                    params.ACTION == 'BUILD_AND_DEPLOY' ||
                    params.ACTION == 'DEPLOY_ONLY'
                }
            }

            steps {

                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ec2-deploy-key',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {

                    sh '''
                        set -e

                        echo "Creating deployment directories..."

                        ssh -i "$SSH_KEY" \
                            -o StrictHostKeyChecking=no \
                            "$SSH_USER@$EC2_HOST" \
                            "mkdir -p ~/microservices/docker/nginx"

                        echo "Copying Docker Compose file..."

                        scp -i "$SSH_KEY" \
                            -o StrictHostKeyChecking=no \
                            docker-compose.yml \
                            "$SSH_USER@$EC2_HOST:~/microservices/docker-compose.yml"

                        echo "Copying Nginx configuration..."

                        scp -i "$SSH_KEY" \
                            -o StrictHostKeyChecking=no \
                            docker/nginx/default.conf \
                            "$SSH_USER@$EC2_HOST:~/microservices/docker/nginx/default.conf"

                        echo "Deploying application on EC2..."

                        ssh -i "$SSH_KEY" \
                            -o StrictHostKeyChecking=no \
                            "$SSH_USER@$EC2_HOST" \
                            "AWS_ACCOUNT_ID='$AWS_ACCOUNT_ID' \
                             AWS_REGION='$AWS_REGION' \
                             IMAGE_TAG='$IMAGE_TAG' \
                             ECR_REGISTRY='$ECR_REGISTRY' \
                             bash -s" <<'REMOTE'

set -e

cd ~/microservices

echo "Creating environment file..."

printf 'AWS_ACCOUNT_ID=%s\\nAWS_REGION=%s\\nIMAGE_TAG=%s\\n' \
    "$AWS_ACCOUNT_ID" \
    "$AWS_REGION" \
    "$IMAGE_TAG" > .env

echo "Checking AWS identity..."

aws sts get-caller-identity

echo "Logging into Amazon ECR..."

aws ecr get-login-password \
    --region "$AWS_REGION" |
docker login \
    --username AWS \
    --password-stdin "$ECR_REGISTRY"

echo "Pulling Docker images..."

docker compose pull

echo "Starting application containers..."

docker compose up -d --remove-orphans

echo "Checking running containers..."

docker compose ps

echo "Deployment completed successfully!"

REMOTE
                    '''
                }
            }
        }

        // ================= DEPLOYMENT VERIFICATION =================

        stage('Deployment Verification') {

            when {
                expression {
                    params.ACTION == 'BUILD_AND_DEPLOY' ||
                    params.ACTION == 'DEPLOY_ONLY'
                }
            }

            steps {

                sh '''
                    set -e

                    echo "Waiting for containers to initialize..."

                    sleep 15

                    echo "Checking User Service..."

                    curl --fail --silent --show-error \
                         --retry 5 \
                         --retry-delay 5 \
                         --retry-connrefused \
                         http://$EC2_HOST:8081/users

                    echo "Checking Product Service..."

                    curl --fail --silent --show-error \
                         --retry 5 \
                         --retry-delay 5 \
                         --retry-connrefused \
                         http://$EC2_HOST:8081/products

                    echo "Checking Order Service..."

                    curl --fail --silent --show-error \
                         --retry 5 \
                         --retry-delay 5 \
                         --retry-connrefused \
                         http://$EC2_HOST:8081/orders

                    echo "All microservices are responding successfully!"
                '''
            }
        }
    }

    // ================= POST ACTIONS =================

    post {

        success {
            echo 'Microservices pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Check Jenkins console output.'
        }

        always {
            echo 'Pipeline execution completed.'
        }
    }
}
