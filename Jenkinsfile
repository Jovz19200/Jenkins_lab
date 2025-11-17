pipeline {
    // We define an agent for each stage, not one for the whole pipeline
    agent none 

    environment {
        DOCKER_IMAGE_NAME = "jovz19200/spring-petclinic" // Your Docker Hub repo
    }

    stages {
        
        stage('Checkout') {
            // This stage just gets the code.
            agent any // Use the default agent just to checkout
            steps {
                echo 'Checking out code from GitHub...'
                checkout scm
                // 'Stash' the code so other stages can use it.
                // This 'saves' the files for the next agents.
                stash(name: 'source', includes: '**/*')
            }
        }

        stage('Build with Maven') {
            // Use a Maven pod as the agent
            agent {
                kubernetes {
                    yaml """
                    apiVersion: v1
                    kind: Pod
                    spec:
                      containers:
                      - name: maven
                        image: maven:3.8-openjdk-17
                        command:
                        - sleep
                        args:
                        - 999999
                    """
                }
            }
            steps {
                // 'maven' is the container name from the YAML above
                container('maven') {
                    // 'Unstash' (get) the saved code from the 'Checkout' stage
                    unstash 'source'
                    echo 'Building the application with Maven...'
                    sh 'mvn clean package'
                    // 'Stash' the results for Sonar and Kaniko
                    // We need source code, pom, and the built .jar
                    stash(name: 'built-files', includes: 'target/*, Dockerfile, k8s/*, pom.xml, src/*')
                }
            }
        }

        // --- 1. NEW STAGE FOR SONARQUBE ANALYSIS ---
        stage('SonarQube Analysis') {
            // We use a Maven agent again because it has Java and the
            // Sonar scanner can run via Maven.
            agent {
                kubernetes {
                    yaml """
                    apiVersion: v1
                    kind: Pod
                    spec:
                      containers:
                      - name: maven
                        image: maven:3.8-openjdk-17
                        command:
                        - sleep
                        args:
                        - 999999
                    """
                }
            }
            steps {
                container('maven') {
                    // Get the built files
                    unstash 'built-files'
                    
                    // Point to the server you named 'SonarQube' in Jenkins config
                    withSonarQubeEnv('SonarQube') {
                        // Run the scanner command
                        sh 'mvn sonar:sonar'
                    }
                }
            }
        }

        // --- 2. NEW STAGE FOR QUALITY GATE ---
        stage('SonarQube Quality Gate') {
            // This stage just waits, so a simple agent is fine
            agent any 
            steps {
                echo "Checking SonarQube Quality Gate..."
                // This step pauses the pipeline and waits for SonarQube
                // to finish its analysis.
                // If the code fails the "Quality Gate", this step
                // will fail the entire pipeline.
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build and Push Image (Kaniko)') {
            // New syntax for the Kaniko pod
            agent {
                kubernetes {
                    yaml """
                    apiVersion: v1
                    kind: Pod
                    spec:
                      containers:
                      - name: kaniko
                        image: gcr.io/kaniko-project/executor-debug:latest
                        command:
                        - sleep
                        args:
                        - 999999
                    """
                }
            }
            steps {
                // 'kaniko' is the container name
                container('kaniko') {
                    // We only need the files for the Docker image
                    unstash 'built-files'
                    
                    echo "Building and pushing image: ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
                    
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKLER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        // Create the config.json file for Kaniko
                        sh "echo '{\"auths\":{\"https://index.docker.io/v1/\":{\"auth\":\"\$(echo -n ${DOCKLER_USER}:${DOCKER_PASS} | base64 -w 0)\"}}}' > /kaniko/.docker/config.json"
                        
                        // Run the Kaniko command
                        sh """
                        /kaniko/executor --context `pwd` \
                                         --dockerfile `pwd`/Dockerfile \
                                         --destination "${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            // New syntax for the kubectl pod
            agent {
                 kubernetes {
                    yaml """
                    apiVersion: v1
                    kind: Pod
                    spec:
                      containers:
                      - name: kubectl
                        image: bitnami/kubectl:latest
                        command:
                        - sleep
                        args:
                        - 999999
                    """
                }
            }
            steps {
                // 'kubectl' is the container name
                container('kubectl') {
                    // We only need the k8s YAML files
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
}