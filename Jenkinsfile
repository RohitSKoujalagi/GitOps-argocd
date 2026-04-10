pipeline {
    agent {
        kubernetes {
            yaml '''
            apiVersion: v1
            kind: Pod
            spec:
              containers:
              - name: trivy-scanner
                image: aquasec/trivy:canary 
                command:
                - sleep
                args:
                - 99d
                volumeMounts:
                - name: docker-sock
                  mountPath: /var/run/docker.sock
              - name: dind-agent
                image: rohitskoujalagi/dind-agent
                command:
                - sleep
                args: 
                - 99d
                volumeMounts:
                - name: docker-sock
                  mountPath: /var/run/docker.sock
              volumes:
              - name: docker-sock
                hostPath:
                  path: /var/run/docker.sock
            '''
        }
    }
    
    environment{
        DOCKER_PAT = credentials("dockerpat-pat")
        GITHUB_REPO = "https://github.com/RohitSKoujalagi/django-helloworld.git" // Update if using your own fork
    }
    
    stages{
        stage("Build & Push Image"){
            steps{
                container("dind-agent"){
                    git url: "${GITHUB_REPO}",
                        branch: "main",
                        credentialsId: 'jenkins-ghid' 
                    
                    echo "Building the Image..."
                    sh "docker build -t django-app:${BUILD_ID} ."
                    sh "docker tag django-app:${BUILD_ID} rohitskoujalagi/django-app:${BUILD_ID}"
                    
                    echo "Pushing to DockerHub..."
                    sh "echo ${DOCKER_PAT} | docker login -u rohitskoujalagi --password-stdin"
                    sh "docker push rohitskoujalagi/django-app:${BUILD_ID}"
                }
            }
        }
        
        stage('Security Scan (Trivy)') {
            steps {
                container("trivy-scanner"){
                    echo "Scanning the image for vulnerabilities..."
                    sh '''
                        docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy:canary image \
                        --severity LOW,MEDIUM,HIGH \
                        rohitskoujalagi/django-app:${BUILD_ID}
                    '''
                }
            }
        }
        
        stage('Update Git Manifests') {
            steps {
                container("dind-agent"){
                    echo "Updating Kubernetes manifests with new image tag..."
                    
                    // 1. Use sed to find and replace the image tag in deployment.yaml
                    sh "sed -i 's|image: rohitskoujalagi/django-app:.*|image: rohitskoujalagi/django-app:${BUILD_ID}|g' k8s/deployment.yaml"
                    

                    withCredentials([gitUsernamePassword(credentialsId: 'jenkins-ghid', gitToolName: 'Default')]) {
                        sh '''
                            # Basic Git configuration (if not already set in Jenkins)
                            git config user.email "jenkins@example.com"
                            git config user.name "Jenkins Build Bot"
                            
                            # Git actions
                            git add .
                            git commit -m "Automated commit from Jenkins Build #${BUILD_ID}"
                            git push origin HEAD:${GITHUB_REPO}
                        '''
                    }
                    // 2. Commit and push the changes back to GitHub
                    // withCredentials([usernamePassword(credentialsId: 'jenkins-ghid', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                    //     sh """
                    //         git config user.email "jenkins@cicd.local"
                    //         git config user.name "Jenkins Pipeline"
                            
                    //         // Using the credentials to authenticate the push
                    //         git remote set-url origin https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/LondheShubham153/django-notes-app.git
                            
                    //         git add k8s/deployment.yaml
                    //         git commit -m "Update image to rohitskoujalagi/django-app:${BUILD_ID}"
                    //         git push origin main
                    //     """
                    // }
                }
            }
        }


    }
}