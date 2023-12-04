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
                        prev_version = sh(script:"sudo git describe --tags --abbrev=0", returnStdout: true).tokenize('-')[1].replace("v","")
                        echo "${prev_version}"
                   }

                   if ("${gitBranch}" == 'master') {
                       echo "****************************************This is master!*********************************"
                       sh "git checkout -b release-${version}"
                       gitBranch = "release-${version}"
                       commitId = sh(returnStdout: true, script: "git log | head -1 | cut -b 7-15")
                       commitId = commitId.substring(1)
                       tagName = "release-v${version}"
                       sh "git tag -a ${tagName} -m 'Version ${version} update'"
                   } else {
                       echo "****************************************${gitBranch}!***********************************"
                       commitId = sh(returnStdout: true, script: "git log | head -1 | cut -b 7-15")
                       commitId = commitId.substring(1)
                       tagName = "release-v${version}"
                       sh "git tag -a ${tagName} -m 'Version ${version} update'"
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
                //    sh "docker push ${dockerRegistry}/super-app-server:${version}"
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
                           from: [type: 'REF', value: "tags/release-v${prev_version}"],
                            to: [type: 'REF', value: "tags/release-v${version}"],
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

※ 정식 버전인 super-app-server:0.4.0 은 QA 검증을 마친 후 CK 및 타본부에 배포될 예정입니다.

${version}의 개선 및 추가된 사항은 아래 Super-App-Server Release Note 링크를 참고 부탁드립니다.

https://flying-balmoral-4aa.notion.site/Super-App-Server-Release-Note-9cb55fc059ef4559988dda2c069e1054

===

Super-App-Server-${version} 버전에서는 다음과 같은 기능이 추가되었습니다.

- Library schema 변경
- ims-312955, 313119, 312293, 314416 해소
- SAS service 가 존재하지 않는 binary를 application으로 배포 지원 (internal controller 단일 배포)
- Service Router에서 SMS 서비스로 라우팅 못 해주던 버그 수정
- 하나의 application에서 여러 controller class 배포 지원
- sasctl을 통한 DataSource 검색 및 적용시, ID/Name 모두 사용할 수 있도록 편의 기능 추가
- sasctl을 통한 DataSource 조회 시, create_at 표기방식 변경

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
                        // to: "dohyun_kim5@tmax.co.kr; ck_rnd1_unit@tmax.co.kr; ck_rnd2_unit@tmax.co.kr; ck_rnd3_unit@tmax.co.kr; ck3_lab@tmax.co.kr; ck_qa_unit@tmax.co.kr;",
                        to: "dohyun_kim5@tmax.co.kr; ck_qa_unit@tmax.co.kr; soohwan_kim@tmax.co.kr; minjae_song@tmax.co.kr; jeongwan_rho@tmax.co.kr; seongmin_lee2@tmax.co.kr; sunghoon_choi@tmax.co.kr; jaehun_lee@tmax.co.kr;",
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
                    sh "git push origin refs/heads/${gitBranch}:refs/heads/${gitBranch}"
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
