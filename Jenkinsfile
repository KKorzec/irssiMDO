pipeline {
    parameters {
        string(name: 'VERSION', defaultValue: '0.0.0', description: '')
        booleanParam(name: 'PROMOTE', defaultValue: true, description: '')
    }
    agent any
    stages {
        stage('Clone') {
            steps {
                sh 'DOCKER_TLS_VERIFY=0 docker rm -f buffer'
                sh 'docker volume prune -f'
                sh 'docker volume create volin'
                sh 'docker run -v volin:/data --name buffer ubuntu'
                sh 'cd ~/ && find irssi || git clone https://github.com/irssi/irssi.git/'
                sh 'docker cp ~/irssi buffer:/data'
                sh 'docker rm buffer'
                echo 'Cloning...'
            }
        }
        stage('Build') {
            steps {
	      checkout([$class: 'GitSCM', 
              branches: [[name: '*/main']], 
              userRemoteConfigs: [[url: 'https://github.com/KKorzec/irssiMDO']]])
              sh 'docker system prune -f'
	      git 'https://github.com/KKorzec/MDOL6'
              sh 'docker build -t irssibld . -f DockerfileBuild'
              sh 'docker volume create volout'
              sh 'docker run --mount type=volume,src="volin",dst=/app --mount type=volume,src="volout",dst=/app/result irssibld bash -c "ls -l && cd irssi && meson setup build && ninja -C build; cp -r ../irssi ../result"'
              echo 'Building...'
               
            }
            post {
                failure {
                    echo 'Build - fail.'
                }
                success {
                    echo 'Build - success.'
                }
            }
        }
        stage('Test') {
            steps {
                sh 'docker build -t irssitst . -f DockerfileTest'
                sh 'docker run -t --mount type=volume,src="volin",dst=/app irssitst bash -c "cd irssi/build && meson test"'
            }
             post {
                failure {
                    echo 'Test - fail.'
                }
                success {
                    echo 'Test - success.'
                }
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploying...'
		sh 'docker rm -f deploybuffer || true'
                sh 'docker run -dit --name deploybuffer --mount type=volume,src="volout",dst=/app/result ubuntu'
                sh 'docker container exec deploybuffer sh -c "apt-get update"'
		sh 'docker container exec deploybuffer sh -c "DEBIAN_FRONTEND="noninteractive" apt-get install -y libglib2.0"'
		sh "docker container exec deploybuffer sh -c 'apt-get install -y libutf8proc-dev'"
		sh "docker container exec deploybuffer sh -c 'cd /app/result/irssi/build/src/fe-text && ./irssi -v'"
		sh "docker container kill deploybuffer"
		sh 'docker rm -f deploybuffer'
            }
            
        }
        stage('Publish') {
            when {
                expression {return params.PROMOTE}
            }
            steps {
                echo 'Publishing...'
		sh 'docker rm -f publishbuffer || true'
                sh 'find /var/jenkins_home/workspace -name "artifacts" || mkdir /var/jenkins_home/workspace/artifacts'
                sh 'docker run -d --rm --name publishbuffer --mount type=volume,src="volout",dst=/app/result --mount type=bind,source=/var/jenkins_home/workspace/artifacts,target=/usr/local/copy ubuntu  bash -c "chmod -R 777 /app && cp -r /app/. /usr/local/copy"'
		sh "touch irssi-ver${params.VERSION}.tar.gz"
		sh "tar --exclude=irssi-ver${params.VERSION}.tar.gz -zcvf irssi-ver${params.VERSION}.tar.gz -C /var/jenkins_home/workspace/artifacts ."
		archiveArtifacts artifacts: "irssi-ver${params.VERSION}.tar.gz"
            }
            
        }
    }
}
