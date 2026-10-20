
pipeline {
    agent any

    tools {
        maven 'maven3'
        jdk 'jdk-21'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        JAVA_TOOL_OPTIONS = '--add-opens=java.base/java.lang=ALL-UNNAMED'
    }

    stages {

        stage('Git Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('Unit Tests') {
            steps {
                sh 'mvn test'
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
                sh 'mkdir -p reports'

                dependencyCheck(
                    odcInstallation: 'DC',
                    additionalArguments: '--scan . --format XML --out reports'
                )

                sh '''
                    echo "Checking Dependency-Check report..."
                    find reports -type f -print
                    test -f reports/dependency-check-report.xml
                '''

                dependencyCheckPublisher(
                    pattern: 'reports/dependency-check-report.xml'
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
                sh '''
                    docker build \
                    -t youngminds73/ekart:latest \
                    -f docker/Dockerfile .
                '''
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
                        set +x
                        echo "$dockerhubpwd" | docker login \
                            -u youngminds73 --password-stdin
                    '''
                }

                sh 'docker push youngminds73/ekart:latest'
            }
        }

        stage('Configure EKS') {
            steps {
                sh '''
                    aws eks update-kubeconfig \
                        --region ap-south-1 \
                        --name project-cluster

                    kubectl get nodes
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl apply -f deploymentservice.yml'
            }
        }
    }
}
