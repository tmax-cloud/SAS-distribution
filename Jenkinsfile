import java.net.*

pipeline {
    agent any
    environment {
//        version = "${params.majorVersion}.${params.minorVersion}.${params.hotfixVersion}"
//        dockerRegistry = "${params.dockerImageRepo}"
        dockerRegistry = "192.168.9.12:5000"

//        publishUrl = "http://${params.publishHost}:${params.publishPort}${params.publishDir}"
//        repoUser = "${params.repoUser}"
//        repoPassword = "${params.repoPassword}"
        publishUrl = "http://192.168.9.12:8081/repository/maven-releases"
        repoUser = "root"
        repoPassword = "tmax@23"

        gitUrl = "${params.gitUrl}"
        gitCred = "${params.gitCred}"
        gitBranch = "${params.gitBranch}"
        version = "${gitBranch.tokenize('-')[1]}"
    }

    stages {
        stage('Git Clone') {
            steps {
                sh 'rm -rf *'
                sh 'rm -rf .g*'
                script {
                    if ("${gitUrl.tokenize('/')[0]}" == 'github.com') {
                        credId = 'dohyun_github'
                    } else {
                        credId = 'gitlab_token'
                    }
                }
                git([branch: "${gitBranch}", credentialsId:"${credId}", url: "http://${gitCred}@${gitUrl}"])
//                   url: "https://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@github.com/dohyunKim12/testing_repo.git"
            }
        }
        stage('Git Fetch') {
            steps {
                sh 'git fetch --all'
//                script {
//                    if ("${gitBranch}" == 'master') {
//                        echo "****************************************This is master!*********************************"
//                        // ToDo master branch면 여기서 checkout -b 로 브랜치 하나 파고(브랜치명은 input version 이용) 체크아웃, 빌드 후 커밋. 태그도 남기기
//                        sh "git checkout -b release-${version}"
//////                        sh "git commit -m 'Packaging for release-${version}'"
//                        sh "git tag release-${version}"
//                        sh "git push --tags"
//                        sh "git push origin"
////                        sh "git push --set-upstream origin release-${version}"
//                        firstBuild = true
//                    } else {
//                        echo "****************************************${gitBranch}!*********************************"
//                        // ToDO 이미 release-0.0.* 브랜치면, 여기서 그냥 제일 최신커밋으로 빌드작업하기.
//                        firstBuild = false
//                    }
//                }
                script {
                    firstBuild = true

                    commitId = sh(returnStdout: true, script: "git log | head -1 | cut -b 7-15")
                    commitId = commitId.substring(1)
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
       stage ('Build and Upload Docker Image') {
           steps {
               script {
                   def version = "${gitBranch}".tokenize('-')[1]
                   def dockerImage = docker.build("${dockerRegistry}/super-app-server:${version}", "--build-arg version=${version} .")
                   docker.withRegistry('', 'dockercred') {
                       dockerImage.push()
                   }
                   sh "docker rmi ${dockerRegistry}/super-app-server:${version}"
               }
           }
       }
       stage('Edit ChangeLog') {
            steps {
                script {
                    def gitDomain = "${gitUrl}".tokenize('/')[0]
                    def changelogString = gitChangelog returnType: 'STRING',
                           from: [type: 'REF', value: 'release-0.0.2'],
//                            to: [type: 'REF', value: 'master'],
                            template:
"""
  {{#tags}}
# {{name}}
 {{#issues}}
 
     {{#ifContainsType commits type='feat'}}
## Features

    {{#commits}}
      {{#ifCommitType . type='feat'}}
**{{#eachCommitScope .}} {{.}} {{/eachCommitScope}}{{{commitDescription .}}}**  *{{authorName}} {{commitTime}}*

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
**{{#eachCommitScope .}} {{.}} {{/eachCommitScope}}{{{commitDescription .}}}**  *{{authorName}} {{commitTime}}*

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
                    writeFile file: "CHANGELOG.md", text: changelogString
                    currentBuild.description = changelogString
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
 안녕하세요. ck1-3팀 김도현입니다.

금주 배포된 super-app-server:${version} release 버전에 대한 안내 및 가이드 메일 드립니다.

${version}의 개선 및 추가된 사항은 첨부된 CHANGELOG.md 파일을 확인 부탁드립니다.

===

※ SuperApp 서비스 예제 프로젝트:
http://gitlab.ck:10081/superobject/super-app-service-example
해당 프로젝트를 참조하여 AbstractServiceObject 를 상속받아 수퍼앱 서비스를 구현하고,
super-app-server.jar 런타임과 같이 실행시키면 테스트가 가능합니다.

구체적인 설치 및 서비스 개발, 그리고 테스트 가이드에 대한 내용은 해당 WIKI 가이드 참고 부탁드립니다.
http://gitlab.ck:10081/superobject/super-object/wikis/home

SuperApp Server 관련된 문의사항 있으실 경우 메일 혹은 WAPL TF를 통해 문의해주시면 바로 대응하도록 하겠습니다.

감사합니다.


- 김도현 드림.

※ SuperApp Server Project :
http://gitlab.ck:10081/superobject/super-object/tree/release-${version}

※ SuperApp Server Runtime :
http://192.168.9.12:8081/repository/maven-releases/com/tmax/super-app-server/${version}/super-app-server-${version}.jar

※ gitlab.ck:10081 접속 방법 :
Default DNS 192.168.1.150 로 설정

""",
                        to: "dohyun_kim5@tmax.co.kr; ck1@tmax.co.kr; ck2@tmax.co.kr; cqa1@tmax.co.kr;",
                        from: "dohyun_kim5@tmax.co.kr"
                )
            }
        }
        stage('Git Push') {
            steps {
                echo "pushing..."
                script {
                    commitMsg = "Release commit - version ${version}"
                    tagName = "release-${version}"
                    sh "git add -A"
                    sh "git commit -m \"${commitMsg}\" || true"
                    sh "git tag -a ${tagName} -m 'Version ${version} update'"
                    sh "git remote rm origin"
                    sh "git remote add origin http://dohyun_kim5:ehgus0303!@${gitUrl}"
                    sh "git remote -v"
                    sh "git push origin refs/tags/${tagName}:refs/tags/${tagName}"
                    sh "git push origin refs/heads/${gitBranch}:refs/heads/${gitBranch}"
                }
            }
        }
    }
}