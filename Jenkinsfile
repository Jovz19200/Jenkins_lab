pipeline {
    agent any // Runs the pipeline on any available Jenkins agent

    environment {
        DOCKER_IMAGE_NAME = "nniyogisubizo/spring-petclinic" 
    }

    stages {
        stage('Checkout') { //
            steps {
                echo 'Checking out code from GitHub...'
                checkout scm
            }
        }

        stage('Build') { //
            steps {
                echo 'Building the application with Maven...'
                // This runs Maven inside a Docker container
                sh 'docker run -v $WORKSPACE:/app -w /app maven:3.8-openjdk-17 mvn clean package'
            }
        }
        
        stage('Test') { //
            steps {
                echo 'Testing (skipping for this lab)...'
                // In a real project, you would run 'mvn test' here
            }
        }
        
        stage('Static Analysis (Optional)') { //
            steps {
                echo 'Skipping SonarQube scan for now.'
            }
        }

        stage('Build Docker Image') {
            steps {
                // The $BUILD_NUMBER is a unique number from Jenkins (e.g., 1, 2, 3)
                echo "Building Docker image: ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
                sh "docker build -t ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER} ."
            }
        }

        stage('Push to Docker Hub') { //
            steps {
                echo "Pushing image to Docker Hub..."
                // Use the 'dockerhub-creds' ID you created in Step 4
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh "docker login -u $DOCKER_USER -p $DOCKER_PASS"
                    sh "docker push ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
                }
            }
        }

        stage('Deploy to Kubernetes') { //
            steps {
                echo "Deploying to Kubernetes..."
                
                // 1. This updates the image in YOUR file: k8s/petclinic.yml
                sh "sed -i 's|image: .*|image: ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}|g' k8s/petclinic.yml"
                
                echo "Applying new Kubernetes config to 'petclinic-prod' namespace..."
                
                // 2. Apply the namespace first (good practice)
                sh "kubectl apply -f k8s/ns-prod.yml"

                // 3. Apply the database and app, specifying the namespace
                sh "kubectl apply -f k8s/db.yml -n petclinic-prod"
                sh "kubectl apply -f k8s/petclinic.yml -n petclinic-prod"
                
                // 4. Check the rollout status in the correct namespace
                sh "kubectl rollout status deployment/petclinic -n petclinic-prod"
            }
        }
    }
}