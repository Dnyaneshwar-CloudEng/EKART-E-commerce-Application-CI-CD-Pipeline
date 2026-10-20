


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
