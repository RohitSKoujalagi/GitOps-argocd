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
                        branch: "master",
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
        
        // stage('Security Scan (Trivy)') {
        //     steps {
        //         container("dind-agent"){
        //             echo "Scanning the image for vulnerabilities..."
        //             sh '''
        //                 docker run --rm \
        //                 -v /var/run/docker.sock:/var/run/docker.sock \
        //                 aquasec/trivy:canary image \
        //                 --severity LOW,MEDIUM,HIGH \
        //                 rohitskoujalagi/django-app:${BUILD_ID}
        //             '''
        //         }
        //     }
        // }
        
        // stage('Update Git Manifests') {
        //     steps {
        //         container("dind-agent"){
        //             echo "Updating Kubernetes manifests with new image tag..."
                    
        //             // 1. Use sed to find and replace the image tag in deployment.yaml
        //             sh "sed -i 's|image: rohitskoujalagi/django-app:.*|image: rohitskoujalagi/django-app:${BUILD_ID}|g' k8s/deployment.yaml"
                    

        //             withCredentials([gitUsernamePassword(credentialsId: 'jenkins-ghid', gitToolName: 'Default')]) {
        //                 sh '''
        //                     # Basic Git configuration (if not already set in Jenkins)
        //                     git config user.email "jenkins@example.com"
        //                     git config user.name "Jenkins Build Bot"
                            
        //                     # Git actions
        //                     git add .
        //                     git commit -m "Automated commit from Jenkins Build #${BUILD_ID}"
        //                     git push origin HEAD:${GITHUB_REPO}
        //                 '''
        //             }
        //             // 2. Commit and push the changes back to GitHub
        //             // withCredentials([usernamePassword(credentialsId: 'jenkins-ghid', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
        //             //     sh """
        //             //         git config user.email "jenkins@cicd.local"
        //             //         git config user.name "Jenkins Pipeline"
                            
        //             //         // Using the credentials to authenticate the push
        //             //         git remote set-url origin https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/LondheShubham153/django-notes-app.git
                            
        //             //         git add k8s/deployment.yaml
        //             //         git commit -m "Update image to rohitskoujalagi/django-app:${BUILD_ID}"
        //             //         git push origin main
        //             //     """
        //             // }
        //         }
        //     }
        // }


        stage('Update Git Manifests') {
            environment {
                // The URL of your SEPARATE ArgoCD manifests repository
                MANIFEST_REPO = "https://github.com/RohitSKoujalagi/GitOps-argocd.git"
                MANIFEST_BRANCH = "HEAD" // or "master" depending on your repo
            }
            steps {
                container("dind-agent") {
                    echo "Cloning ArgoCD manifests repo to update image tag..."
                    
                    // We wrap the whole shell script in the credentials binding 
                    // so the 'git clone' and 'git push' are both authenticated
                    withCredentials([gitUsernamePassword(credentialsId: 'jenkins-ghid', gitToolName: 'Default')]) {
                        
                        // Use triple double-quotes (""") so Groovy evaluates variables like ${BUILD_ID}
                        sh """
                            # 1. Clean up any previous runs to avoid conflicts
                            rm -rf manifests-repo-dir
                            
                            # 2. Clone the SECOND repository
                            git clone ${MANIFEST_REPO} manifests-repo-dir
                            
                            # 3. Enter the new repository directory
                            cd manifests-repo-dir
                            
                            # 4. Update the image tag (Make sure the path matches your repo structure!)
                            sed -i 's|image: rohitskoujalagi/django-app:.*|image: rohitskoujalagi/django-app:${BUILD_ID}|g' k8s/deployment.yaml
                            
                            # 5. Set up the Git Bot identity
                            git config user.email "jenkins@example.com"
                            git config user.name "Jenkins Build Bot"
                            
                            # 6. Add, Commit, and Push
                            git add .
                            git commit -m "Automated image update to build #${BUILD_ID} [skip ci]"
                            
                            # Git knows 'origin' is the MANIFEST_REPO because we just cloned it
                            git push origin HEAD:${MANIFEST_BRANCH}
                        """
                    }
                }
            }
        }


    }
}