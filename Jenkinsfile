pipeline {
    agent any

    parameters {
        string(name: 'registry', defaultValue: 'dtr.nagarro.com:443');
        string(name: 'username', defaultValue: 'girish');
        string(name: 'appPort',defaultValue:'8080');
        string(name: 'dockerPort', defaultValue: '6000');
        string(name: 'nodePort', defaultValue: '30157');

    }

    tools {
        maven 'Maven3'
        jdk 'Java'
    }
    options {
        timestamps()

        timeout(time: 1, unit: 'HOURS')

        skipDefaultCheckout()

        buildDiscarder(logRotator(daysToKeepStr: '10', numToKeepStr: '10'))

        disableConcurrentBuilds()
    }
    stages {
        stage('Checkout') {
            steps {
                echo "*************************** Checkout master **************************"
                checkout scm
            }
        }
        stage('Build') {
            steps {
                echo "*************************** maven clean**************************"
                sh "mvn clean install"
            }
        }
        stage('Unit Testing') {
            steps {
                echo "*************************** Test Case**************************"
                sh "mvn test"
            }
        }
        stage ('Sonar Analysis') {
            steps {
                echo "*************************** Sonar Analysis**************************"
                withSonarQubeEnv("Test_Sonar")
                {
                    sh "mvn sonar:sonar"
                }
            }
        }
        stage ('Upload to Artifactory') {
            steps {
                echo "*************************** aupload artifactory**************************"
                rtMavenDeployer (
                    id: 'deployer',
                    serverId: '123456789@artifactory',
                    releaseRepo: 'CI-Automation-JAVA',
                    snapshotRepo: 'CI-Automation-JAVA'
                )
                rtMavenRun (
                    pom: 'pom.xml',
                    goals: 'clean install',
                    deployerId: 'deployer'
                )
                rtPublishBuildInfo (
                    serverId: '123456789@artifactory'
                )
            }
       }
        stage('Docker Image') {
            steps {
                echo "*************************** Docker image build **************************"
                sh "docker build -t ${params.registry}/i_${params.username}_master:${BUILD_NUMBER} --no-cache -f Dockerfile ."
            }
        }
		stage ('Push to DTR') {
            steps {
			echo "*************************** Docker image push to repo **************************"
                	sh "docker push ${params.registry}/i_${params.username}_master:${BUILD_NUMBER}"
            }
        }
        
        stage('Docker deployment') { 
            steps {
                echo '*************************** Docker run container **************************'
                sh "docker run --name c_${params.username}_master -d -p ${params.dockerPort}:${params.appPort} ${params.registry}/i_${params.username}_master:${BUILD_NUMBER}"
            }
        }
        stage ('Helm Chart Deployment') {
        	steps {
                echo '*************************** helm deploy **************************'
        	    withKubeConfig([credentialsId: '1ad89c30-9317-4883-af32-e6678c370e35', serverUrl: 'https://demo-dns-3ca02a47.hcp.westus2.azmk8s.io:443']) {
                    sh "kubectl create ns ${params.username}-master-${BUILD_NUMBER}"
        		    sh "helm install helm-chart-master helm-chart --set image=${params.registry}/i_${params.username}_master:${BUILD_NUMBER} --set nodeport=${params.nodePort} --namespace=${params.username}-master-${BUILD_NUMBER}"
        	    }
        	}
        }

    }
}