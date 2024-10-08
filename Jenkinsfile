pipeline {

    agent any 
    tools {
        jdk 'jdk'
        nodejs 'nodejs'
    }
    environment  {
        SCANNER_HOME=tool 'sonar-scanner'
        ProjectID = 'clear-practice-430609-e0'
        // AWS_ACCOUNT_ID = credentials('ACCOUNT_ID')
        CR_REPO_NAME = credentials('CR_REPO2')
        // AWS_DEFAULT_REGION = 'us-east-1'
        // REPOSITORY_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/"
    }
    stages {
        stage('Cleaning Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git credentialsId: 'GITHUB', url: 'https://github.com/maniimanjari/End-to-End-Kubernetes-Three-Tier-DevSecOps-Project.git'
            }
        }
        // stage('Sonarqube Analysis') {
        //     steps {
        //         dir('Application-Code/backend') {
        //             withSonarQubeEnv('sonar-server') {
        //                 sh ''' $SCANNER_HOME/bin/sonar-scanner \
        //                 -Dsonar.projectName=three-tier-backend \
        //                 -Dsonar.projectKey=three-tier-backend '''
        //             }
        //         }
        //     }
        // }
        // stage('Quality Check') {
        //     steps {
        //         script {
        //             waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token' 
        //         }
        //     }
        // }
        // stage('OWASP Dependency-Check Scan') {
        //     steps {
        //         dir('Application-Code/backend') {
        //             dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
        //             dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
        //         }
        //     }
        // }
        stage('Trivy File Scan') {
            steps {
                dir('Application-Code/backend') {
                    sh 'trivy fs . > trivyfs.txt'
                }
            }
        }
        stage("Docker Image Build") {
            steps {
                script {
                    dir('Application-Code/backend') {
                            sh 'docker system prune -f'
                            sh 'docker container prune -f'
                            sh 'docker build -t ${CR_REPO_NAME} .'
                    }
                }
            }
        }
        stage("ECR Image Pushing") {
            steps {
                script {
                        withCredentials([file(credentialsId: "jenkins123-key", variable: 'FILE')]) {
                            sh "gcloud auth activate-service-account jenkins123@clear-practice-430609-e0.iam.gserviceaccount.com --key-file=${FILE} --project=${ProjectID}"
                            sh "gcloud auth configure-docker us-central1-docker.pkg.dev --quiet ;\
                                docker push us-central1-docker.pkg.dev/backend;"
                        }
                        // sh 'aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${REPOSITORY_URI}'
                        // sh 'docker tag ${CR_REPO_NAME} ${REPOSITORY_URI}${CR_REPO_NAME}:${BUILD_NUMBER}'
                        // sh 'docker push ${REPOSITORY_URI}${CR_REPO_NAME}:${BUILD_NUMBER}'
                }
            }
        }
        stage("TRIVY Image Scan") {
            steps {
                sh 'trivy image ${REPOSITORY_URI}${CR_REPO_NAME}:${BUILD_NUMBER} > trivyimage.txt' 
            }
        }
        stage('Checkout Code') {
            steps {
                git credentialsId: 'GITHUB', url: 'https://github.com/maniimanjari/End-to-End-Kubernetes-Three-Tier-DevSecOps-Project.git'
            }
        }
        stage('Update Deployment file') {
            environment {
                GIT_REPO_NAME = "End-to-End-Kubernetes-Three-Tier-DevSecOps-Project"
                GIT_USER_NAME = "maniimanjari"
            }
            steps {
                dir('Kubernetes-Manifests-file/Backend') {
                    withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
                        sh '''
                            git config user.email "maniimanjari@gmail.com"
                            git config user.name "maniimanjari"
                            BUILD_NUMBER=${BUILD_NUMBER}
                            echo $BUILD_NUMBER
                            imageTag=$(grep -oP '(?<=backend:)[^ ]+' deployment.yaml)
                            echo $imageTag
                            sed -i "s/${CR_REPO_NAME}:${imageTag}/${CR_REPO_NAME}:${BUILD_NUMBER}/" deployment.yaml
                            git add deployment.yaml
                            git commit -m "Update deployment Image to version \${BUILD_NUMBER}"
                            git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:master
                        '''
                    }
                }
            }
        }
    }
}