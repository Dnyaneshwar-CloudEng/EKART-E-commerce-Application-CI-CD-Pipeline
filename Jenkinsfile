pipeline {
    agent any

    tools {
        maven 'maven3'
        jdk 'jdk-8'
    }

    environment {
        // Must match the SonarQube Scanner name in Manage Jenkins -> Tools
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('git checkout') {
            steps {
                // Uses the same repo/branch the Jenkinsfile was loaded from
                checkout scm
            }
        }

        stage('compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('unit tests') {
            steps {
                sh 'mvn test'
            }
        }

        stage('SonarQube analysis') {
            // SonarScanner needs Java 11+, so this stage uses JDK 21
            tools { jdk 'jdk-21' }
            steps {
                withSonarQubeEnv('sonar-scanner') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=EKART \
                        -Dsonar.projectName=EKART \
                        -Dsonar.java.binaries=target/classes
                    """
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck(
                    odcInstallation: 'DC',
                    additionalArguments: '--format XML'
                )

                sh '''
                    echo "Checking Dependency-Check reports..."
                    find . -type f -name "dependency-check-report.xml"
                '''

                dependencyCheckPublisher(
                    pattern: '**/dependency-check-report.xml'
                )
            }
        }

        stage('Build') {
            steps {
                sh 'mvn package -DskipTests=true'
            }
        }

        stage('deploy to Nexus') {
            steps {
                withMaven(
                    globalMavenSettingsConfig: 'global-maven',
                    jdk: 'jdk-8',
                    maven: 'maven3',
                    mavenSettingsConfig: '',
                    traceability: true
                ) {
                    sh 'mvn deploy -DskipTests=true'
                }
            }
        }

        stage('build and Tag docker image') {
            steps {
                sh 'docker build -t youngminds73/ekart:latest -f docker/Dockerfile .'
            }
        }

        stage('Push image to Hub') {
            steps {
                withCredentials([
                    string(credentialsId: 'dockerhub-pwd', variable: 'dockerhubpwd')
                ]) {
                    sh '''
                        echo "$dockerhubpwd" | docker login \
                            -u youngminds73 --password-stdin
                    '''
                }

                sh 'docker push youngminds73/ekart:latest'
            }
        }

        stage('EKS and Kubectl configuration') {
            steps {
                sh 'aws eks update-kubeconfig --region ap-south-1 --name project-cluster'
            }
        }

        stage('Deploy to k8s') {
            steps {
                sh 'kubectl apply -f deploymentservice.yml'
            }
        }
    }
}
