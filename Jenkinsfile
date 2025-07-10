import java.text.SimpleDateFormat

def TODAY = (new SimpleDateFormat("yyyyMMddHHmmss")).format(new Date())
pipeline {
    agent any
    environment {
        strDockerTag = "${TODAY}_${BUILD_ID}"
        strDockerImage ="yullmm/cicd_jenkins:${strDockerTag}"
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'master', url:'https://github.com/yullmm0531/jenkins.git'
            }
        }
        stage('Build') {
            steps {
                sh './mvnw clean install'
            }
        }
        stage('Unit Test') {
            steps {
                sh './mvnw test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/TEST-*.xml'
                }
            }
        }
        stage('Docker Image Build') {
            steps {
                script {
                    //oDockImage = docker.build(strDockerImage)
                    oDockImage = docker.build(strDockerImage, "--build-arg VERSION=${strDockerTag} -f Dockerfile .")
                }
            }
        }
        stage('Docker Image Push') {
            steps {
                script {
                    docker.withRegistry('', 'DockerHub_Credential') {
                        oDockImage.push()
                    }
                }
            }
        }
        stage('Staging Deploy') {
            steps {
                sshagent(credentials: ['Staging-PrivateKey']) {
                    sh "ssh -o StrictHostKeyChecking=no root@3.36.249.211 docker container rm -f guestbookapp"
                    sh "ssh -o StrictHostKeyChecking=no root@3.36.249.211 docker container run \
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
            steps {
                // sh '~/lab/sw/jmeter/bin/jmeter.sh -j jmeter.save.saveservice.output_format=xml -n -t src/main/jmx/guestbook_loadtest.jmx -l loadtest_result.jtl'
                // perfReport filterRegex: '', showTrendGraphs: true, sourceDataFiles: 'loadtest_result.jtl'
                echo "JMeter LoadTest"
            }
        }
    }
    post {
        always {
            slackSend(tokenCredentialId: 'slack-token'
                , channel: '#교육'
                , color: 'good'
                , message: "${JOB_NAME} (${BUILD_NUMBER}) YULLMM 빌드가 끝났습니다. Details: (<${BUILD_URL} | here >)")
        }
        success {
            slackSend(tokenCredentialId: 'slack-token'
                , channel: '#교육'
                , color: 'good'
                , message: "${JOB_NAME} (${BUILD_NUMBER}) YULLMM 빌드가 성공적으로 끝났습니다. Details: (<${BUILD_URL} | here >)")
        }
        failure {
            slackSend(tokenCredentialId: 'slack-token'
                , channel: '#교육'
                , color: 'danger'
                , message: "${JOB_NAME} (${BUILD_NUMBER}) YULLMM 빌드가 실패하였습니다. Details: (<${BUILD_URL} | here >)")
        }
    }
}
