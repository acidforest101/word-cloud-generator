pipeline {
  agent none
  options {
    timestamps ()
  }
  stages {
    stage ('Create container for WCG') {
      agent {
        dockerfile { filename 'Dockerfile1'
                     args '--privileged'
        }
      }
      stages {
        stage('Get GO tools. Lint and check') { 
          steps { sh '''
                  GOPATH=$WORKSPACE
                  export PATH=&quot;$PATH:$(go env GOPATH)/bin&quot;
                  go get github.com/tools/godep
                  go get github.com/smartystreets/goconvey
                  go get github.com/GeertJohan/go.rice/rice
                  go get github.com/acidforest101/word-cloud-generator/wordyapi
                  go get github.com/gorilla/mux
                  '''
          }
        }
        stage('Make lint and check') {
          steps {
            sh 'make lint'
            sh 'make check'
          }
        }
        stage('Building code and uploading artifacts to Nexus') { 
          steps {sh '''
	         export GOPATH=$WORKSPACE
                 export PATH="$PATH:$(go env GOPATH)/bin"
                 rm -f artifacts/*
                 sed -i 's/1.DEVELOPMENT/1.$BUILD_NUMBER/g' ./rice-box.go
                 GOOS=linux GOARCH=amd64 go build -o ./artifacts/word-cloud-generator -v .
                 gzip ./artifacts/word-cloud-generator
                 mv ./artifacts/word-cloud-generator.gz ./artifacts/word-cloud-generator
                 ls ./artifacts/
                 '''
                 nexusArtifactUploader artifacts: [[artifactId: 'word-cloud-generator', classifier: '', file: 'artifacts/word-cloud-generator', type: 'gz']], credentialsId: 'upload', groupId: 'webapp', nexusUrl: '192.168.66.11:8081', nexusVersion: 'nexus3', protocol: 'http', repository: 'word-cloud-generator', version: '1.$BUILD_NUMBER'
                 }
               }
             }
           }
           stage('Integration tests') {
             agent {
               dockerfile { filename 'Dockerfile2'
                            dir 'alpine'
                            args '--privileged'
               }
             }
             stages {
               stage ('Download artifact from vault and start it') {
                 steps { sh '''
                         curl -X GET -u downloader:downloader "http://192.168.66.11:8081/repository/word-cloud-generator/webapp/word-cloud-generator/1.$BUILD_NUMBER/word-cloud-generator-1.$BUILD_NUMBER.gz" -o /opt/wordcloud/word-cloud-generator.gz
                         gunzip -f /opt/wordcloud/word-cloud-generator.gz
                         rm -f artifacts/*
                         chmod +x /opt/wordcloud/word-cloud-generator
                         /opt/wordcloud/word-cloud-generator
                         sleep 5
                         '''
                 }
               }
               stage ('Running integration tests') {
                 steps { sh '''
		         res=`curl -s -H "Content-Type: application/json" -d '{"text":"ths is a really really really important thing this is"}' http://127.0.0.1:8888/version | jq '. | length'`
                         if [ "1" != "$res" ]; then exit 99; fi
                         res=`curl -s -H "Content-Type: application/json" -d '{"text":"ths is a really really really important thing this is"}' http://127.0.0.1:8888/api | jq '. | length'`
                         if [ "7" != "$res" ]; then exit 99; fi
                         '''
                 }
               }
             }
           }
         }
}
