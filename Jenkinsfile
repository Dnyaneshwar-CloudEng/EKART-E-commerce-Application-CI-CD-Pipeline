
pipeline {
    agent any

    tools {
        maven 'maven3'
        jdk 'jdk-21'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Git Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Unit Tests') {
            steps {
                sh 'mvn test -DskipTests=true'
            }
        }

        stage('SonarQube Analysis') {
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

        stage('Deploy to Nexus') {
            steps {
                withMaven(
                    globalMavenSettingsConfig: 'global-maven',
                    jdk: 'jdk-21',
                    maven: 'maven3',
                    mavenSettingsConfig: '',
                    traceability: true
                ) {
                    sh 'mvn deploy -DskipTests=true'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t youngminds73/ekart:latest -f docker/Dockerfile .'
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'dockerhub-pwd',
                        variable: 'dockerhubpwd'
                    )
                ]) {
                    sh '''
                        echo "$dockerhubpwd" | docker login \
                            -u youngminds73 --password-stdin
                    '''
                }

                sh 'docker push youngminds73/ekart:latest'
            }
        }

        stage('Configure EKS') {
            steps {
                sh 'aws eks update-kubeconfig --region ap-south-1 --name project-cluster'
                sh 'kubectl get nodes'
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl apply -f deploymentservice.yml'
            }
        }
    }
}
