pipeline {
    // We will define an agent for each stage, not one for the whole pipeline
    agent none 

    environment {
        // This variable is available to all stages
        DOCKER_IMAGE_NAME = "nniyogisubizo/spring-petclinic" 
    }

    stages {
        
        stage('Checkout') {
            // This stage just gets the code.
            agent any // Use any available agent just to checkout
            steps {
                echo 'Checking out code from GitHub...'
                checkout scm
                // 'Stash' the code so other stages can use it.
                // This 'saves' the files for the next agents.
                stash(name: 'source', includes: '**/*')
            }
        }

        stage('Build with Maven') {
            // Use a Maven agent just to build the .jar file
            agent {
                docker { image 'maven:3.8-openjdk-17' }
            }
            steps {
                // 'Unstash' (get) the saved code from the 'Checkout' stage
                unstash 'source'
                echo 'Building the application with Maven...'
                sh 'mvn clean package'
                // 'Stash' the results (the .jar, Dockerfile, and k8s files)
                // for the next stages.
                stash(name: 'built-files', includes: 'target/spring-petclinic-*.jar, Dockerfile, k8s/*')
            }
        }

        stage('Build and Push Image (Kaniko)') {
            // This is the fix: Use a Kaniko agent to build the image
            agent {
                docker { image 'gcr.io/kaniko-project/executor-debug:latest' }
            }
            steps {
                // Get the saved files from the 'Build' stage
                unstash 'built-files'
                
                echo "Building and pushing image: ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
                
                // Kaniko needs Docker Hub credentials in a special config file.
                // This 'withCredentials' block securely creates that file.
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKLER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    // Create the config.json file for Kaniko
                    sh "echo '{\"auths\":{\"https://index.docker.io/v1/\":{\"auth\":\"\$(echo -n ${DOCKLER_USER}:${DOCKER_PASS} | base64 -w 0)\"}}}' > /kaniko/.docker/config.json"
                    
                    // Run the Kaniko command to build and push the image
                    sh """
                    /kaniko/executor --context `pwd` \
                                     --dockerfile `pwd`/Dockerfile \
                                     --destination "${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            // Use an agent that has 'kubectl' and 'sed'
            agent {
                docker { image 'bitnami/kubectl:latest' }
            }
            steps {
                // Get the saved files again
                unstash 'built-files'
                
                echo "Deploying to Kubernetes..."
                
                // This command updates your k8s/petclinic.yml file with the new image
                sh "sed -i 's|image: .*|image: ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}|g' k8s/petclinic.yml"
                
                echo "Applying new Kubernetes config to 'petclinic-prod' namespace..."
                
                // Apply all your k8s files
                sh "kubectl apply -f k8s/ns-prod.yml"
                sh "kubectl apply -f k8s/db.yml -n petclinic-prod"
                sh "kubectl apply -f k8s/petclinic.yml -n petclinic-prod"
                
                // Check the rollout status
                sh "kubectl rollout status deployment/petclinic -n petclinic-prod"
            }
        }
    }
}





// pipeline {
//     agent any // Runs the pipeline on any available Jenkins agent

//     environment {
//         DOCKER_IMAGE_NAME = "nniyogisubizo/spring-petclinic" 
//     }

//     stages {
//         stage('Checkout') { //
//             steps {
//                 echo 'Checking out code from GitHub...'
//                 checkout scm
//             }
//         }

//         stage('Build') { //
//             steps {
//                 echo 'Building the application with Maven...'
//                 // This runs Maven inside a Docker container
//                 sh 'docker run -v $WORKSPACE:/app -w /app maven:3.8-openjdk-17 mvn clean package'
//             }
//         }
        
//         stage('Test') { //
//             steps {
//                 echo 'Testing (skipping for this lab)...'
//                 // In a real project, you would run 'mvn test' here
//             }
//         }
        
//         stage('Static Analysis (Optional)') { //
//             steps {
//                 echo 'Skipping SonarQube scan for now.'
//             }
//         }

//         stage('Build Docker Image') {
//             steps {
//                 // The $BUILD_NUMBER is a unique number from Jenkins (e.g., 1, 2, 3)
//                 echo "Building Docker image: ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
//                 sh "docker build -t ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER} ."
//             }
//         }

//         stage('Push to Docker Hub') { //
//             steps {
//                 echo "Pushing image to Docker Hub..."
//                 // Use the 'dockerhub-creds' ID you created in Step 4
//                 withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
//                     sh "docker login -u $DOCKER_USER -p $DOCKER_PASS"
//                     sh "docker push ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
//                 }
//             }
//         }

//         stage('Deploy to Kubernetes') { //
//             steps {
//                 echo "Deploying to Kubernetes..."
                
//                 // 1. This updates the image in YOUR file: k8s/petclinic.yml
//                 sh "sed -i 's|image: .*|image: ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}|g' k8s/petclinic.yml"
                
//                 echo "Applying new Kubernetes config to 'petclinic-prod' namespace..."
                
//                 // 2. Apply the namespace first (good practice)
//                 sh "kubectl apply -f k8s/ns-prod.yml"

//                 // 3. Apply the database and app, specifying the namespace
//                 sh "kubectl apply -f k8s/db.yml -n petclinic-prod"
//                 sh "kubectl apply -f k8s/petclinic.yml -n petclinic-prod"
                
//                 // 4. Check the rollout status in the correct namespace
//                 sh "kubectl rollout status deployment/petclinic -n petclinic-prod"
//             }
//         }
//     }
// }



