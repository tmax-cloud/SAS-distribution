pipeline {
    agent any 
    environment {
      // gitlab
        gitUrl = "192.168.1.150:10081/superobject/super-object.git"
        gitCred = "Rbxxb7pBtyw6D_KqbNWa"
        gitBranch = "${params.gitBranch}"
        version = "${params.version}"
        prev_version = "${params.prev_version}"
        dockerRegistry = "192.168.9.12:5000"
        publishUrl = "http://192.168.9.12:8081/repository/maven-releases"
        repoUser = "root"
        repoPassword = "tmax@23"
    }

    stages {
        stage('Git Clone') {
            steps {
                sh "sudo chown -R jenkins ."
                sh 'ls -al'
                sh 'rm -rf *'
                sh 'rm -rf .g*'
                sh 'git init'
                script {
                    echo "${env.WORKSPACE}"
                    if ("${gitUrl.tokenize('/')[0]}" == 'github.com') {
                        credId = 'dohyun_github'
                    } else {
                        credId = 'gitlab_token'
                    }
                    git([branch: "${gitBranch}", credentialsId:"${credId}", url: "http://${gitCred}@${gitUrl}"])
                }
            }
        }
        stage('Git Tagging') {
            steps {
                sh 'git fetch --all'
               script {
                   if ("${prev_version}" == "default") {
                        prev_version = sh(script:"sudo git describe --tags --abbrev=0", returnStdout: true).tokenize('-')[1]
                        echo "${prev_version}"
                   }

                   if ("${gitBranch}" == 'master') {
                       echo "****************************************This is master!*********************************"
                       sh "git checkout -b release-${version}"
                       commitId = sh(returnStdout: true, script: "git log | head -1 | cut -b 7-15")
                       commitId = commitId.substring(1)
                       tagName = "release-${version}"
                       sh "git tag -a ${tagName} -m 'Version ${version} update'"
                   } else {
                       echo "****************************************${gitBranch}!***********************************"
                       commitId = sh(returnStdout: true, script: "git log | head -1 | cut -b 7-15")
                       commitId = commitId.substring(1)
                       tagName = "release-${version}"
                      //  sh "git tag -a ${tagName} -m 'Version ${version} update'"
                   }
               }
            }
        }
       stage('Build Jar') {
           steps {
               echo "${version}"
               sh 'chmod +x ./gradlew'
               sh "./gradlew clean build jenkins -PbuildVersion=${version} -PcommitId=${commitId}"
           }
       }
       stage('Upload Jar') {
           steps {
               sh "./gradlew publish -PbuildVersion=${version} -PpublishUrl=${publishUrl} -PrepoUser=${repoUser} -PrepoPassword=${repoPassword}"
           }
       }
       stage('Build Package & Upload to ftp server') {
           steps {
                sh "sudo sh ./scripts/packaging.sh"
           }
       }
       stage ('Build and Upload Docker Image') {
           steps {
               script {
                   def dockerImage = docker.build("${dockerRegistry}/super-app-server:${version}", "--build-arg version=${version} .")
                   docker.withRegistry('', 'dockercred') {
                       dockerImage.push()
                   }
                   sh "docker login hyperregistry.tmaxcloud.org -u admin -p admin"
                   sh "docker tag ${dockerRegistry}/super-app-server:${version} hyperregistry.tmaxcloud.org/super-app-server/super-app-server:${version}"
                   sh "docker push hyperregistry.tmaxcloud.org/super-app-server/super-app-server:${version}"
                   sh "docker rmi ${dockerRegistry}/super-app-server:${version}"
               }
           }
       }
       stage('Edit ChangeLog') {
            steps {
                script {

                    def gitDomain = "${gitUrl}".tokenize('/')[0]
                    def changelogString = gitChangelog returnType: 'STRING',
                           from: [type: 'REF', value: "tags/release-${prev_version}"],
                            to: [type: 'REF', value: "tags/release-${version}"],
                            template:
"""
  {{#tags}}
# {{name}}
 {{#issues}}
 
     {{#ifContainsType commits type='feat'}}
## Features

    {{#commits}}
      {{#ifCommitType . type='feat'}}
**{{#eachCommitScope .}} {{.}} {{/eachCommitScope}}{{{commitDescription .}}}**  ([{{hash}}](http://${gitDomain}/{{ownerName}}/{{repoName}}/commit/{{hash}})) *{{authorName}} {{commitTime}}*

{{#messageBodyItems}}
  *{{.}}* 
{{/messageBodyItems}}

      {{/ifCommitType}}
    {{/commits}}
  {{/ifContainsType}} 
  
     {{#ifContainsType commits type='mod'}}
## Refactor

    {{#commits}}
      {{#ifCommitType . type='mod'}}
**{{#eachCommitScope .}} {{.}} {{/eachCommitScope}}{{{commitDescription .}}}**  ([{{hash}}](http://${gitDomain}/{{ownerName}}/{{repoName}}/commit/{{hash}})) *{{authorName}} {{commitTime}}*

{{#messageBodyItems}}
  *{{.}}*
{{/messageBodyItems}}

      {{/ifCommitType}}
    {{/commits}}
  {{/ifContainsType}} 
  
     {{#ifContainsType commits type='fix'}}
## Bug Fixes

    {{#commits}}
      {{#ifCommitType . type='fix'}}
**{{#eachCommitScope .}} {{.}} {{/eachCommitScope}}{{{commitDescription .}}}**  ([{{hash}}](http://${gitDomain}/{{ownerName}}/{{repoName}}/commit/{{hash}})) *{{authorName}} {{commitTime}}*

{{#messageBodyItems}}
  *{{.}}*
{{/messageBodyItems}}

      {{/ifCommitType}}
    {{/commits}}
  {{/ifContainsType}} 
  
     {{#ifContainsType commits type='etc'}}
## OTHERS

    {{#commits}}
      {{#ifCommitType . type='etc'}}
**{{#eachCommitScope .}} {{.}} {{/eachCommitScope}}{{{commitDescription .}}}**  ([{{hash}}](http://${gitDomain}/{{ownerName}}/{{repoName}}/commit/{{hash}})) *{{authorName}} {{commitTime}}*

{{#messageBodyItems}}
  *{{.}}* 
{{/messageBodyItems}}

      {{/ifCommitType}}
    {{/commits}}
  {{/ifContainsType}} 
 {{/issues}}
{{/tags}}
"""
                    writeFile file: "tmp/CHANGELOG_new", text: changelogString
                    currentBuild.description = changelogString
                    sh "mv CHANGELOG.md tmp/tmpfile"
                    sh "cat tmp/CHANGELOG_new > CHANGELOG.md"
                    sh "cat tmp/tmpfile >> CHANGELOG.md"
                    sh "ls -al"
                }
            }
       }
        stage('Send Email') {
            steps {
                emailext (
                        attachmentsPattern: 'CHANGELOG.md',
                        subject: "[super-app-server] Release Notes - super-app-server:${version}",
                        body:
                                """
 안녕하세요. ck1-2팀 김도현입니다.

금주 배포된 super-app-server:${version} release 버전에 대한 안내 및 가이드 메일 드립니다.

${version}의 개선 및 추가된 사항은 아래 Super-App-Server Release Note 링크를 참고 부탁드립니다.

https://flying-balmoral-4aa.notion.site/Super-App-Server-Release-Note-9cb55fc059ef4559988dda2c069e1054

===

Super-App-Server-${version} 버전에서는 다음과 같은 기능이 추가되었습니다.

- **Common**
    - Master 이중화 지원
        - SASWorker 기동 시 env 로 `SUB_MASTER_IP` 지정 (asset/yaml 참고)
        - Active Master down 시 worker는 SUB Master (Standby) 로 연결 시도
    - SAS Default Logger 리팩토링 - Root 로거 형식 변경, TaskObject, SAS Service 에 대해 trace-id, span-id 추가
    - Application Active-Standby 모드 지원
    (App deploy 시 `-l option` 으로 지정 가능, default : Active-Active)
    - Application scale 기능 삭제
    (App 배포 시 WorkerPool 에 존재하는 모든 WorkerSAS에 App 배포)
    - SAG 재기동 시 SASMaster에서 URL routing rule 재전송 로직 추가
    - asset/yaml 설정값 추가 (ENCODING, TIMEZONE, PORT, etc)
    - ClassLoader refactoring - multiple app jar 들을 한번에 묶어 ClassGraph scan하도록 구조변경
- **SASCTL**
    - `start` command → `deploy` 로 변경
    - 바이너리 upload 와 동시에 workerPool 배포 기능 삭제
    (app upload → app deploy 순차적 명령어 실행)
    - delete 명령어 버그 수정
- **Controller**
    - External Controller Active-Standby 모드 지원
- **Custom Gateway**
    - CGW 재기동 시 복구 지원
    - HTTP URI query param 및 path variable 설정 지원

자세한 예시 코드 및 가이드를 Wiki에 업로드 할 예정이오니
super-object Wiki를 참고해 주시면 감사하겠습니다.

===

※ SuperApp 서비스 예제 프로젝트:
http://gitlab.ck:10081/superobject/super-app-service-example
해당 프로젝트를 참조하여 AbstractServiceObject 를 상속받아 슈퍼앱 서비스를 구현하고,
super-app-runtime.jar 런타임을 실행시키면 테스트가 가능합니다.

구체적인 설치 및 서비스 개발, 그리고 테스트 가이드에 대한 내용은 해당 WIKI 가이드 참고 부탁드립니다.
http://gitlab.ck:10081/superobject/super-object/wikis/home

SuperApp Server 관련된 문의사항 있으실 경우 메일 혹은 WAPL TF를 통해 문의해주시면 바로 대응하도록 하겠습니다.

감사합니다.


- 김도현 드림.


※ SuperApp Server Runtime :
http://192.168.9.12/binary/super-app-runtime/super-app-runtime-${version}

※ SuperApp Server Maven Repository :
http://192.168.9.12:8081/#browse/browse:maven-releases:com%2Ftmax%2Fsuper-app-server%2F0.0.5%2Fsuper-app-server-${version}.jar

※ SuperApp Server Project :
http://gitlab.ck:10081/superobject/super-object/tree/release-${version}

※ SuperApp Server Container Image :
hyperregistry.tmaxcloud.org/super-app-server/super-app-server:${version}

※ gitlab.ck:10081 접속 방법 :
Default DNS 192.168.1.150 로 설정

""",
                        to: "dohyun_kim5@tmax.co.kr; ck1@tmax.co.kr; ck2@tmax.co.kr; ck3@tmax.co.kr; ck3_lab@tmax.co.kr; cqa1@tmax.co.kr;",
                        from: "dohyun_kim5@tmax.co.kr"
                )
            }
        }
        stage('Git Push') {
            steps {
                echo "pushing..."
                script {
                    commitMsg = "Release commit - version ${version}"
                    sh "git add -A"
                    sh "git commit -m \"${commitMsg}\" || true"
                    sh "git remote rm origin"
                    sh "git remote add origin http://dohyun_kim5:ehgus0303!@${gitUrl}"
                    sh "git remote -v"
                    sh "git push origin refs/tags/${tagName}:refs/tags/${tagName}"
                    sh "git push origin refs/heads/release-${version}:refs/heads/release-${version}"
                }
            }
        }
        stage('Cleaning...') {
            steps {
                echo "All work Done. Cleaning..."
                echo "Check the @tmp directory for confirm."
                script {
                    sh "sudo echo ${WORKSPACE}"
                    sh "sudo rsync -a ${env.WORKSPACE} ${env.WORKSPACE}@tmp"
                    sh "sudo echo ${WORKSPACE}"
                    sh "cd ${WORKSPACE}"
                    sh "sudo chown -R jenkins ."
                }
                deleteDir()
            }
        }
    }
}
