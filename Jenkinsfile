pipeline {
    agent any

    environment {
        GIT_REPO_URL = 'https://github.com/Nithin2751/assignment-mvn.git'
        SONAR_URL = 'http://18.234.44.116:30900'
        SONAR_CRED_ID = 'sonar-cred'
        NEXUS_URL = 'http://18.234.44.116:30001/repository/sample-releases'
        NEXUS_DOCKER_REPO = '18.234.44.116:30002'
        NEXUS_CREDENTIAL_ID = 'nexus-cred'
        MAX_BUILDS_TO_KEEP = 5
    }

    tools {
        jdk 'java-21'
        maven 'maven-3.9.10'
    }

    stages {
        stage('Checkout') {
            steps {
                git url: "${GIT_REPO_URL}", branch: 'master'
            }
        }

        stage('Create Sonar Project') {
            steps {
                script {
                    def projectName = "${env.JOB_NAME}-${env.BUILD_NUMBER}".replace('/', '-')
                    withCredentials([string(credentialsId: "${SONAR_CRED_ID}", variable: 'SONAR_TOKEN')]) {
                        sh """
                        curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer $SONAR_TOKEN" -X POST \
                        "${SONAR_URL}/api/projects/create?project=${projectName}&name=${projectName}" || true
                        """
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def projectName = "${env.JOB_NAME}-${env.BUILD_NUMBER}".replace('/', '-')
                    withCredentials([string(credentialsId: "${SONAR_CRED_ID}", variable: 'SONAR_TOKEN')]) {
                        sh """
                        mvn clean verify sonar:sonar \
                            -Dsonar.projectKey=${projectName} \
                            -Dsonar.host.url=${SONAR_URL} \
                            -Dsonar.login=${SONAR_TOKEN}
                        """
                    }
                }
            }
        }

        stage('Build and Tag Artifact') {
            steps {
                script {
                    sh "mvn clean package -DskipTests"
                    def artifactName = "my-app-${BUILD_NUMBER}.jar"
                    sh """
                        mkdir -p tagged-artifacts
                        cp target/*.jar tagged-artifacts/${artifactName}
                        echo "Artifact tagged as ${artifactName}"
                    """
                }
            }
        }

        stage('Push Artifact to Nexus') {
            steps {
                script {
                    def version = "1.0.${BUILD_NUMBER}"
                    def artifactId = "my-app"
                    def groupPath = "com/mycompany/app"
                    def nexusPath = "${groupPath}/${artifactId}/${version}"
                    def finalArtifact = "${artifactId}-${version}.jar"

                    sh """
                        mv tagged-artifacts/my-app-${BUILD_NUMBER}.jar tagged-artifacts/${finalArtifact}
                    """

                    withCredentials([usernamePassword(credentialsId: "${NEXUS_CREDENTIAL_ID}", usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                        sh """
                        curl -u $NEXUS_USER:$NEXUS_PASS \
                             --upload-file tagged-artifacts/${finalArtifact} \
                             ${NEXUS_URL}/${nexusPath}/${finalArtifact}
                        """
                    }
                }
            }
        }

        stage('Create Dockerfile') {
            steps {
                script {
                    writeFile file: 'Dockerfile', text: """
                    FROM openjdk:21-jdk-slim
                    WORKDIR /app
                    COPY tagged-artifacts/my-app-*.jar app.jar
                    EXPOSE 8080
                    ENTRYPOINT ["java", "-jar", "app.jar"]
                    """
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def imageTag = "${NEXUS_DOCKER_REPO}/my-app:${BUILD_NUMBER}"
                    sh "docker build -t ${imageTag} ."
                }
            }
        }

        stage('Push Docker Image to Nexus') {
            steps {
                script {
                    def imageTag = "${NEXUS_DOCKER_REPO}/my-app:${BUILD_NUMBER}"
                     withCredentials([usernamePassword(credentialsId: "${NEXUS_CREDENTIAL_ID}", usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    sh """
                        echo $NithinDevops@1 | docker login ${NEXUS_DOCKER_REPO} -u $USERNAME --password-stdin

                        docker push ${imageTag}
                        docker logout ${NEXUS_DOCKER_REPO}
                        """
                    }
                }
            }
        }

        stage('Delete Old Sonar Projects') {
            steps {
                script {
                    def currentBuildNum = env.BUILD_NUMBER.toInteger()
                    def minBuildToKeep = currentBuildNum - MAX_BUILDS_TO_KEEP.toInteger()

                    if (minBuildToKeep > 0) {
                        withCredentials([string(credentialsId: "${SONAR_CRED_ID}", variable: 'SONAR_TOKEN')]) {
                            for (int i = 1; i <= minBuildToKeep; i++) {
                                def oldProject = "${env.JOB_NAME}-${i}".replace('/', '-')
                                echo "Deleting old Sonar project: ${oldProject}"
                                sh """
                                curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer $SONAR_TOKEN" -X POST \
                                  "${SONAR_URL}/api/projects/delete" \
                                  -d "project=${oldProject}" || true
                                """
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
