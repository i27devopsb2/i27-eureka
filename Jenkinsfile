// This Jenkinsfile is for the Eureka Deployment.
pipeline {
    agent {
        label 'k8s-slave'
    }
    environment {
        APPLICATION_NAME = "eureka"
        POM_VERSION = readMavenPom().getVersion()
        POM_PACKAGING = readMavenPom().getPackaging()
        //version+ packaging
        DOCKER_HUB = "docker.io/i27devopsb2"
        DOCKER_CREDS = credentials('i27devopsb2_docker_creds')
        SONAR_URL = "http://34.122.97.102:9000"
        SONAR_TOKEN = credentials('sonar_creds')
    }
    tools {
        maven 'Maven-3.8.8'
        jdk 'JDK-17'
    }
    stages {
        stage ('Build'){
            // Application Build happens here
            steps { // jenkins env variable no need of env 
                echo "Building the ${env.APPLICATION_NAME} application"
                sh "mvn clean package -DskipTests=true"
                //-DskipTests=true 
            }
        }
        stage ('Unit Tests') {
            steps {
                echo "Performing Unit tests for ${env.APPLICATION_NAME} application"
                sh "mvn test"
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
        stage ('Sonar') {
            steps {
                echo "Starting Sonarqube With Quality Gates"
                withSonarQubeEnv('SonarQube'){ // manage jenkins > configure  > sonarqube scanner
                    sh """
                        mvn clean verify sonar:sonar \
                            -Dsonar.projectKey=i27-eureka \
                            -Dsonar.host.url=${env.SONAR_URL} \
                            -Dsonar.login=${SONAR_TOKEN}
                    """
                }
                timeout (time: 2, unit: 'MINUTES') { // NANOSECONDS, SECONDS , MINUTES , HOURS, DAYS
                    script {
                        waitForQualityGate abortPipeline: true
                    }
                } 

            }
        }
        stage ('Docker Format') {
            steps {
                // Tell me, how can i read a pom.xml from jenkinfile
                echo "Actual Format: ${env.APPLICATION_NAME}-${env.POM_VERSION}-${env.POM_PACKAGING}"
                // need to have below formating 
                // eureka-buildnumber-brnachname.paackaging
                //eureka-06-master.jar
                echo "Custom Format: ${env.APPLICATION_NAME}-${currentBuild.number}-${BRANCH_NAME}.${env.POM_PACKAGING}"
            }
        }
        stage ('Docker Build and Push') {
            steps {
                // doker build -t name: tag 
                sh """
                  ls -la
                  cp ${workspace}/target/i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} ./.cicd
                  ls -la ./.cicd
                  echo "******************************** Build Docker Image ********************************"
                  docker build --force-rm --no-cache --pull --rm=true --build-arg JAR_SOURCE=i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} -t ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT} ./.cicd
                  docker images
                  echo "******************************** Login to Docker Repo ********************************"
                  docker login -u ${DOCKER_CREDS_USR} -p ${DOCKER_CREDS_PSW}
                  echo "******************************** Docker Push ********************************"
                  docker push ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}
                """
            }
        }
        stage ('Deploy to Dev') {
            steps {
                echo "******************************** Deploying to Dev Environment ********************************"
                withCredentials([usernamePassword(credentialsId: 'maha_docker_vm_creds', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
                    // some block
                    // With the help of this block, ,the slave will be connecting to docker-vm and execute the commands to create the containers.
                    //sshpass -p ssh -o StrictHostKeyChecking=no user@host command_to_run
                    //sh "sshpass -p ${PASSWORD} -v ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} hostname -i" 
                    
                    // Pull the image on the Docker Server
                    sh "sshpass -p ${PASSWORD} -v ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker pull ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
                    
                    try {
                        // Stop the Container
                        echo "Stoping the Container"
                        sh "sshpass -p ${PASSWORD} -v ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker stop ${env.APPLICATION_NAME}-dev"

                        // Remove the Container 
                        echo "Removing the Container"
                        sh "sshpass -p ${PASSWORD} -v ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker rm ${env.APPLICATION_NAME}-dev"
                    } catch(err) {
                        echo "Caught the Error: $err"
                    }

                    // Create a Container 
                    echo "Creating the Container"
                    sh "sshpass -p ${PASSWORD} -v ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker run -d -p 5761:8761 --name ${env.APPLICATION_NAME}-dev ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
                }
            }
        }
    }
}

// cp /home/i27k8s10/jenkins/workspace/i27-Eureka_master/target/i27-eureka-0.0.1-SNAPSHOT.jar ./.cicd

// workspace/target/i27-eureka-0.0.1-SNAPSHOT-jar

// i27devopsb2/eureka:tag


// Eureka container runs at 8761 port 
// I will configure env's in a way they will have diff host ports
// dev ==> 5761 (HP)
// test ==> 6761 (HP)
// stage ==> 7761 (HP)
// Prod ==> 8761 (HP)


// stop ==> remove 

// run