#!groovy

pipeline {

	agent any
	
    parameters {
        string(name: 'localPath', defaultValue: '/Users/oscuro/workspace/commitconf2018/income-predictor-data', description: 'Local path of income predictor data')
    }

    environment {
        ORG_NAME = "oscurorestalion"
        APP_NAME = "income-predictor-ml"
        APP_CONTEXT_ROOT = "oscuroweb"
        CONTAINER_NAME = "ci-${APP_NAME}"
        IMAGE_NAME = "${ORG_NAME}/${APP_NAME}"
    }

    stages {
        stage('Compile') {
		    agent {
		        docker {
		            image 'maven:3.5.4-jdk-8'
		            args '--network ci --mount type=volume,source=ci-maven-home,target=/root/.m2'
		        }
		    }
            steps {
                echo "-=- compiling project -=-"
                sh "mvn clean compile"
            }
        }

//        stage('Unit tests') {
//            steps {
//                echo "-=- execute unit tests -=-"
//                sh "mvn dependency:tree -Dverbose -Dincludes=oscuroweb"
//                sh "mvn test"
//            }
//        }

        stage('Package') {
		    agent {
		        docker {
		            image 'maven:3.5.4-jdk-8'
		            args '--network ci --mount type=volume,source=ci-maven-home,target=/root/.m2'
		        }
		    }
            steps {
                echo "-=- packaging project -=-"
                sh "mvn package -DskipTests"
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }
        stage('Get input data') {
            steps {
                echo "-=- getting income input data -=-"
                sh "curl https://archive.ics.uci.edu/ml/machine-learning-databases/adult/adult.data >> adult.data"
                archiveArtifacts artifacts: 'adult.data', fingerprint: true
            }
        }

        stage('Build Docker image') {
            steps {
                echo "-=- build Docker image -=-"
                script {
                    step ([
                    	$class: "CopyArtifact",
                 		projectName: "${JOB_NAME}",
                 		filter : "target/*.jar",
                 		selector: [$class: "SpecificBuildSelector", buildNumber: "${BUILD_NUMBER}"]
             		])
             		
                    def image = docker.build("${IMAGE_NAME}:${env.BUILD_ID}")
				}            
			}
        }

        stage('Run Docker image') {
            steps {
                echo "-=- run Docker image -=-"
                script {
                    step ([
                    	$class: "CopyArtifact",
                 		projectName: "${JOB_NAME}",
                 		filter: "adult.data",
                 		target: "/data/income-predictor/input-data/",
                 		selector: [$class: "SpecificBuildSelector", buildNumber: "${BUILD_NUMBER}"]
             		])
         		}
                sh "docker run -p 8081:8081 --network ci -v ${localPath}:/data/income-predictor -e MODEL_PATH=/data/income-predictor/output-data -e DATASET_PATH=/data/income-predictor/input-data/ --name ${env.CONTAINER_NAME} -d ${IMAGE_NAME}:${env.BUILD_ID}"
            }
        }

        stage('Integration tests') {
            steps {
                echo "-=- execute integration tests -=-"
                echo "Not an executable project so no integration test phase needed"
            }
        }

        stage('Performance tests') {
            steps {
                echo "-=- execute performance tests -=-"
                echo "Not an executable project so no performance test phase needed"
            }
        }

        stage('Dependency vulnerability tests') {
		    agent {
		        docker {
		            image 'maven:3.5.4-jdk-8'
		            args '--network ci --mount type=volume,source=ci-maven-home,target=/root/.m2'
		        }
		    }
            steps {
                echo "-=- run dependency vulnerability tests -=-"
                sh "mvn dependency-check:check"
            }
        }

        stage('Push Artifact') {
            steps {
                echo "-=- push Artifact -=-"
                withCredentials([usernamePassword(credentialsId: 'docker-hub', passwordVariable: 'DOCKERHUB_PASSWORD', usernameVariable: 'DOCKERHUB_USERNAME')]) {
                    sh "docker login -u ${DOCKERHUB_USERNAME} -p ${DOCKERHUB_PASSWORD}"
                    sh "docker push ${IMAGE_NAME}:${env.BUILD_ID}"
                }
            }
        }
    }

    post {
        always {
            echo "-=- remove deployment -=-"
            echo "Not an executable project so no Docker image needed"
        }
    }
}