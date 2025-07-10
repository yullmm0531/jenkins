import java.text.SimpleDateFormat

def TODAY = (new SimpleDateFormat("yyyyMMddHHmmss")).format(new Date())

pipeline {
    agent { label 'master' }
    environment {
        strDockerTag = "${TODAY}_${BUILD_ID}"
        strDockerImage ="yullmm/cicd_jenkins:${strDockerTag}"
    }

    stages {
        stage('Checkout') {
            agent { label 'agent1' }
            steps {
                git branch: 'master', url:'https://github.com/yullmm0531/jenkins.git'
            }
        }
        stage('Build') {
            agent { label 'agent1' }
            steps {
                sh './mvnw clean package'
            }
        }
        stage('Unit Test') {
            agent { label 'agent1' }
            steps {
                sh './mvnw test'
            }
            
            post {
                always {
                    junit '**/target/surefire-reports/TEST-*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            agent { label 'agent1' }
            steps{
                echo 'SonarQube Analysis'
                /*
                withSonarQubeEnv('SonarQube-Server'){
                    sh '''
                        ./mvnw sonar:sonar \
                        -Dsonar.projectKey=guestbook \
                        -Dsonar.host.url=http://192.168.56.143:9000 \
                        -Dsonar.login=21193ff67973f0efc068ac33ce547e3da8c671b7
                    '''
                }
                */
            }
        }
        stage('SonarQube Quality Gate'){
            agent { label 'agent1' }
            steps{
                echo 'SonarQube Quality Gate'
                /*
                timeout(time: 1, unit: 'MINUTES') {
                    script{
                        def qg = waitForQualityGate()
                        if(qg.status != 'OK') {
                            echo "NOT OK Status: ${qg.status}"
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        } else{
                            echo "OK Status: ${qg.status}"
                        }
                    }
                }
                */
            }
        }
        stage('Docker Image Build') {
            agent { label 'agent2' }
            steps {
                git branch: 'master', url:'https://github.com/yullmm0531/jenkins.git'
                sh './mvnw clean package'
                script {
                    //oDockImage = docker.build(strDockerImage)
                    oDockImage = docker.build(strDockerImage, "--build-arg VERSION=${strDockerTag} -f Dockerfile .")
                }
            }
        }
        stage('Docker Image Push') {
            agent { label 'agent2' }
            steps {
                script {
                    docker.withRegistry('', 'DockerHub_Credential') {
                        oDockImage.push()
                    }
                }
            }
        }
        stage('Staging Deploy') {
            agent { label 'master' }
            steps {
                sshagent(credentials: ['Staging-PrivateKey']) {
                    sh "ssh -o StrictHostKeyChecking=no root@3.34.200.85 docker container rm -f guestbookapp"
                    sh "ssh -o StrictHostKeyChecking=no root@3.34.200.85 docker container run \
                                        -d \
                                        -p 38080:80 \
                                        --name=guestbookapp \
                                        -e MYSQL_IP=3.36.249.211 \
                                        -e MYSQL_PORT=3303 \
                                        -e MYSQL_DATABASE=guestbook \
                                        -e MYSQL_USER=root \
                                        -e MYSQL_PASSWORD=education \
                                        ${strDockerImage} "
                }
            }
        }
        stage ('JMeter LoadTest') {
            agent { label 'agent1' }
            steps { 
                sh '~/lab/sw/jmeter/bin/jmeter.sh -j jmeter.save.saveservice.output_format=xml -n -t src/main/jmx/guestbook_loadtest.jmx -l loadtest_result.jtl' 
                perfReport filterRegex: '', showTrendGraphs: true, sourceDataFiles: 'loadtest_result.jtl' 
            } 
        }
    }
    post { 
        always { 
            emailext (attachLog: true, body: '본문', compressLog: true
                    , recipientProviders: [buildUser()], subject: '제목', to: 'yu3papa.j@gmail.com')

        }
        success { 
            slackSend(tokenCredentialId: 'slack-token'
                , channel: '#교육'
                , color: 'good'
                , message: "${JOB_NAME} (${BUILD_NUMBER}) 빌드가 성공적으로 끝났습니다. Details: (<${BUILD_URL} | here >)")
        }
        failure { 
            slackSend(tokenCredentialId: 'slack-token'
                , channel: '#교육'
                , color: 'danger'
                , message: "${JOB_NAME} (${BUILD_NUMBER}) 빌드가 실패하였습니다. Details: (<${BUILD_URL} | here >)")
    }
  }
}
