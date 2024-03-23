// This Jenkinsfile is for the Eureka Deployment.
pipeline {
    agent {
        label 'k8s-slave'
    }
    parameters {
        choice(name: 'buildOnly',
            choices: 'no\nyes',
            description: 'This will only build the application'
        )
        choice(name: 'scanOnly',
            choices: 'no\nyes',
            description: 'This will Scan the application'
        )
        choice(name: 'dockerPush',
            choices: 'no\nyes',
            description: 'This will build the app, docker build, docker push'
        )
        choice(name: 'deployToDev',
            choices: 'no\nyes',
            description: 'This will Deploy the app to Dev env'
        )
        choice(name: 'deployToTest',
            choices: 'no\nyes',
            description: 'This will Deploy the app to Test env'
        )
        choice(name: 'deployToStage',
            choices: 'no\nyes',
            description: 'This will Deploy the app to Stage env'
        )
        choice(name: 'deployToProd',
            choices: 'no\nyes',
            description: 'This will Deploy the app to Prod env'
        )
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
            when {
                anyOf {
                    expression {
                        params.buildOnly == 'yes'
                       // params.dockerPush == 'yes'
                    }
                }
            }
            // Application Build happens here
            steps { // jenkins env variable no need of env 
                script {
                    buildApp().call()
                }

                //-DskipTests=true 
            }
        }
        stage ('Unit Tests') {
            when {
                anyOf {
                    expression {
                        params.buildOnly == 'yes'
                        params.dockerPush == 'yes'
                    }
                }
            }
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
            when {
                    expression {
                        params.scanOnly == 'yes'  
                    }
            }
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
        /*
        stage ('Docker Format') {
            steps {
                // Tell me, how can i read a pom.xml from jenkinfile
                echo "Actual Format: ${env.APPLICATION_NAME}-${env.POM_VERSION}-${env.POM_PACKAGING}"
                // need to have below formating 
                // eureka-buildnumber-brnachname.paackaging
                //eureka-06-master.jar
                echo "Custom Format: ${env.APPLICATION_NAME}-${currentBuild.number}-${BRANCH_NAME}.${env.POM_PACKAGING}"
            }
        }*/
        stage ('Docker Build and Push') {
            when {
                anyOf {
                    expression {
                        params.dockerPush == 'yes'
                    }
                }
            }
            steps {
                // doker build -t name: tag 
                script {
                    dockerBuildandPush().call()
                }

            }
        }
        stage ('Deploy to Dev') {
            when {
                expression {
                    params.deployToDev == 'yes'
                }
            }
            steps {
                script {
                    imageValidation().call()
                    dockerDeploy('dev', '5761' , '8761').call()
                    echo "Deployed to Dev Succesfully!!!!"
                }
            }
        }
        stage ('Deploy to Test') {
            when {
                expression {
                    params.deployToTest == 'yes'
                }
            }
            steps {
                script {
                    imageValidation().call()
                    echo "***** Entering Test Environment *****"
                    dockerDeploy('tst', '6761', '8761').call()
                }
            }
        }
        stage ('Deploy to Stage') {
            when {
                expression {
                    params.deployToStage == 'yes'
                }
            }
            steps {
                script {
                    imageValidation().call()
                    dockerDeploy('stage', '7761', '8761').call()
                }
            }
        } 
        stage ('Deploy to Prod') {
            when {
                // deployToProd === yes "and" branch "release/*****" 
                allOf {
                    anyOf {
                        expression {
                            params.deployToProd == 'yes'
                        }
                    }
                    anyOf {
                        branch 'release/*'
                    }
                }
            }
            steps {
                script {
                    imageValidation().call()
                    dockerDeploy('prod', '8761', '8761').call()
                }
            }
        }
        stage ('clean'){
            steps {
                cleanWs()
            }
        }
    }
}

// This method will build image and push to registry
def dockerBuildandPush(){
    return {
            echo "******************************** Build Docker Image ********************************"
            sh "cp ${workspace}/target/i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} ./.cicd"
            sh "ls -la ./.cicd"
            sh "docker build --force-rm --no-cache --pull --rm=true --build-arg JAR_SOURCE=i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} -t ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT} ./.cicd"
            echo "******************************** Login to Docker Repo ********************************"
            sh "docker login -u ${DOCKER_CREDS_USR} -p ${DOCKER_CREDS_PSW}"
            echo "******************************** Docker Push ********************************"
            sh "docker push ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
            echo "Pushed the image succesfully!!!"
    }
}

// This method is developed for Deploying our App in different environments
def dockerDeploy(envDeploy, hostPort, contPort) {
    return {
    echo "******************************** Deploying to $envDeploy Environment ********************************"
    withCredentials([usernamePassword(credentialsId: 'maha_docker_vm_creds', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
        // some block
        // With the help of this block, ,the slave will be connecting to docker-vm and execute the commands to create the containers.
        //sshpass -p ssh -o StrictHostKeyChecking=no user@host command_to_run
        //sh "sshpass -p ${PASSWORD} -v ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} hostname -i" 
        
    script {
        // Pull the image on the Docker Server
        sh "sshpass -p ${PASSWORD} -v ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker pull ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
        
        try {
            // Stop the Container
            echo "Stoping the Container"
            sh "sshpass -p ${PASSWORD} -v ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker stop ${env.APPLICATION_NAME}-$envDeploy"

            // Remove the Container 
            echo "Removing the Container"
            sh "sshpass -p ${PASSWORD} -v ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker rm ${env.APPLICATION_NAME}-$envDeploy"
             } catch(err) {
            echo "Caught the Error: $err"
        }

        // Create a Container 
        echo "Creating the Container"
        sh "sshpass -p ${PASSWORD} -v ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker run -d -p $hostPort:$contPort --name ${env.APPLICATION_NAME}-$envDeploy ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
        }
    }
    }
    
}

def imageValidation() {
    return {
        println ("Pulling the docker image")
        try {
        sh "docker pull ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}" 
        }
        catch (Exception e) {
            println("OOPS!, docker images with this tag is not available")
            buildApp().call()
            dockerBuildandPush().call()
        }
    }
}


def buildApp() {
    return {    
        echo "Building the ${env.APPLICATION_NAME} application"
        sh "mvn clean package -DskipTests=true"
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
