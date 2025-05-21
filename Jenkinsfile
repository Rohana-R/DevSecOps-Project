def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger'
    ]
pipeline{
    agent any
    environment {
        GIT_CRED = 'git-ssh'
        SCANNER_HOME=tool 'sonar-scanner'
        TMDB_V3_API_KEY = credentials('tmdb-api-key')
        IMAGE_NAME = "rohana1234/netflix" // Name of the image created in Jenkins
        // CONTAINER_NAME1 = "netflix1" // Name of the container created in Jenkins
        // CONTAINER_NAME2 = "netflix2"
        // CONTAINER_NAME3 = "netflix3"
        CONTAINERS = 'container1,container2,container3'
        PORTS = '8082,8083,8084'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'release', url: 'git@github.com:Rohana-R/DevSecOps-Project.git', credentialsId: 'git-ssh'            
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=DevSecOps-Project \
                    -Dsonar.projectKey=DevSecOps-Project'''
                }
            }
        }
       
        stage('OWASP FS SCAN') {
             steps {
             withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
            dependencyCheck additionalArguments: "--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey ${NVD_API_KEY}", odcInstallation: 'DP-Check'
             }
            dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
       }

        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"     
            }
        }
        // stage('Clean Up Docker Resources') {
        //     steps {
        //         script {
        //             // Remove the specific container
        //             sh '''
        //             if docker ps -a --format '{{.Names}}' | grep -q $CONTAINER_NAME; then
        //                 echo "Stopping and removing container: $CONTAINER_NAME"
        //                 docker stop $CONTAINER_NAME
        //                 docker rm $CONTAINER_NAME
        //             else
        //                 echo "Container $CONTAINER_NAME does not exist."
        //             fi
        //             '''

        //             // Remove the specific image
        //             sh '''
        //             if docker images -q $IMAGE_NAME; then
        //                 echo "Removing image: $IMAGE_NAME"
        //                 docker rmi -f $IMAGE_NAME
        //             else
        //                 echo "Image $IMAGE_NAME does not exist."
        //             fi
        //             '''
        // //         }
        // //     }
        // // }
        stage('Clean Up Docker Resources') {
            steps {
                script {
                    def containerList = env.CONTAINERS.split(',')

                    for (c in containerList) {
                        sh """
                        if docker ps -a --format '{{.Names}}' | grep -q ^${c}\$; then
                            echo "Stopping and removing container: ${c}"
                            docker stop ${c}
                            docker rm ${c}
                        else
                            echo "Container ${c} does not exist."
                        fi
                        """
                    }

                    sh """
                    if docker images -q $IMAGE_NAME; then
                        echo "Removing image: $IMAGE_NAME"
                        docker rmi -f $IMAGE_NAME
                    else
                        echo "Image $IMAGE_NAME does not exist."
                    fi
                    """
                }
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker-cred'){   
                       sh 'docker build --build-arg TMDB_V3_API_KEY=$TMDB_V3_API_KEY -t $IMAGE_NAME .'
                       sh 'docker push $IMAGE_NAME'
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image $IMAGE_NAME > trivyimage.txt"
            }
        }
        stage('Deploy to container') {
    steps {
        script {
            def containerList = env.CONTAINERS.split(',')         // e.g., web1,web2,web3
            def portList = env.PORTS.split(',')                   // e.g., 8082,8083,8084

            for (int i = 0; i < containerList.size(); i++) {
                def container = containerList[i]
                def port = portList[i]
                sh "docker run -d --name ${container} -p ${port}:80 ${IMAGE_NAME}"
            }
        }
    }
}
    }
post {
     failure {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
                "Build Number: ${env.BUILD_NUMBER}<br/>" +
                "URL: ${env.BUILD_URL}<br/>",
            to: 'rohana.r.90@gmail.com',                               
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }

        always {
            echo 'slack Notification.'
            slackSend channel: '#multicont-netflix',
            color: COLOR_MAP [currentBuild.currentResult],
            message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URl}"
            
        }
    }
}
