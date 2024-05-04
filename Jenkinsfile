pipeline {
//Nexus will not be used in this project

    agent any
    // most of the stages below in this pipeline will run on Jenkins server itself without any agent except
    // for the last stage Kubernetes Deploy stage which will run on the kops-project14-EC2 that has kops installedd
    // to provision the k8s cluster on AWS2
/*
	tools {
        maven "maven3"
    }
*/
    environment {

        // Remove all NEXUS env variables.

        registry = "dave123456789/vproappdock"
        //my docker hub account

        registryCredential = 'dockerhub'
        // registryCredential is the ID of the dockerhub login credentials for my repository configured in Jenkins Credentials
        // The ID is in the first column of the main Credentials table in Jenkins.

        //ARTVERSION = "${BUILD_ID}"
    }

    stages{
        // do not need to clone source code
        stage('BUILD'){
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('UNIT TEST'){
            steps {
                sh 'mvn test'
            }
        }

        stage('INTEGRATION TEST'){
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage ('CODE ANALYSIS WITH CHECKSTYLE'){
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }



        stage('CODE ANALYSIS with SONARQUBE') {
            // my SONARSCANNER is sonarscanner as defined in Jenkins Tools
            // my SONARSERVER is sonarserver as defined in Jenkins System

            environment {
                scannerHome = tool 'sonarscanner'
            }

            steps {
                withSonarQubeEnv('sonarserver') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile-repo \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }

                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }




        //DOCKER:
        stage('Building Docker App image') {
            steps{
              script {
                // registry as defined above
                // tag is the BUILD_NUMBER + will concatenate build and BUILD_NUMBER
                dockerImage = docker.build registry + ":$BUILD_NUMBER"
              }
            }
        }
        
        stage('Deploy and Upload Image to docker hub') {
        // upload image to docker hub
          steps{
            script {
              docker.withRegistry( '', registryCredential ) {
                dockerImage.push("$BUILD_NUMBER")
                // this tag of image will be pushed
                dockerImage.push('latest')
                // also push the latest
              }
            }
          }
        }

        stage('Remove Unused docker image') {
          steps{
            sh "docker rmi $registry:$BUILD_NUMBER"
            // this removes local image on Jenkins server
          }
        }




        // KUBERNETES:
        stage('Kubernetes Deploy') {
	        agent { label 'KOPS' }
            // the kops-project14-EC2 instance will be configured as a slave node to run the shell command below
            // the label KOPS was used in the Jenkins slave node configuration for the kops-project14-EC2 instance along with a lot of other setup.
            // Using SSH to the master to node connection. Security groups reprovisioned on AWS2.

            // Helm needs to be installed on the kops-project14-EC2 instance to run the charts from the source code and configure the k8s cluster
            // there is no helm plugin for Jenkins, but it is not required in this case anyways.
            // https://helm.sh/docs/intro/quickstart/
            // https://helm.sh/docs/intro/install/
            // https://helm.sh/docs/intro/quickstart/#initialize-a-helm-chart-repository
            // https://github.com/helm/helm/releases

            // Note that we are bringing up the k8s infra (1 master and 2 worker nodes) by using kops command line from this kops-project14-EC2 instance
            // This can be automated by we are doing it manually for this project
            // the kops commands for this setup are in the README file.  The s3 bucket is on AWS2 us-east-1 region vprofile-kops-s3-state-project14
            // the cluster name is kops-project14.********.com (my own public domain)
            // very that the hosted zone is up and running prior to running the kops, in Route53
            // the kops public private key pair (rsa-gen) are in the ~/.ssh directory on the kops-project14-EC2 instance
            // Include the public key in the kops create cluster command if you wish to SSH into the master/nodes once the infra is up.

            //appimage variable will be passed into the vproappdep.yml file and referenced as {{ .Values.appimage}}
            // when Jenkinspipeline invokes helm command to provison the k8s cluster.

            //created namespace as prod with kubectl create namespace prod
            // charts will be deployed in prod namespace.
            
            steps {
                    sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:${BUILD_NUMBER} --namespace prod"
            }
        }

    }
    //Stages block end


    post {
        always {
            echo 'Slack Notifications.'


            // devops2-co workspace. Second workspce.  Create a new channel for project20 called #devop23-projecct20
            slackSend channel: '#devop23-project20',
                color: COLOR_MAP[currentBuild.currentResult],
                //failOnError: true, 
                // BUILD_NUMBER, JOB_NAME and BUILD_URL are all Jenkins pipeline env vars
                // We will be tagging the docker image with the Jenkins BUILD_NUMBER per  dockerImage = docker.build registry + ":$BUILD_NUMBER"
                message: "*${currentBuild.currentResult}:* Pipeline Job ${env.JOB_NAME} Pipeline Docker Build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
        }
    }

}
//Pipeline block end
