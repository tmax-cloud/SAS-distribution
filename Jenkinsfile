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
//         stage('Send Email') {
//             steps {
//                 emailext (
//                         attachmentsPattern: 'CHANGELOG.md',
//                         subject: "[super-app-server] Release Notes - super-app-server:${version}",
//                         body:
//                                 """
//  ???????????????. ck1-2??? ??????????????????.

// ?????? ????????? super-app-server:${version} release ????????? ?????? ?????? ??? ????????? ?????? ????????????.

// ${version}??? ?????? ??? ????????? ????????? ????????? CHANGELOG.md ????????? ?????? ??????????????????.

// ===

// Super-App-Server-0.1.3 ??????????????? ????????? ?????? ????????? ?????????????????????.

// - ?????? ????????? application delete ??????
// - ????????? ????????? application upload??? ????????? ????????? application apply

// ????????? ?????? ?????? ??? ???????????? Wiki??? ????????? ??? ???????????????
// super-object Wiki??? ????????? ????????? ?????????????????????.

// ===

// ??? SuperApp ????????? ?????? ????????????:
// http://gitlab.ck:10081/superobject/super-app-service-example
// ?????? ??????????????? ???????????? AbstractServiceObject ??? ???????????? ????????? ???????????? ????????????,
// super-app-runtime.jar ???????????? ??????????????? ???????????? ???????????????.

// ???????????? ?????? ??? ????????? ??????, ????????? ????????? ???????????? ?????? ????????? ?????? WIKI ????????? ?????? ??????????????????.
// http://gitlab.ck:10081/superobject/super-object/wikis/home

// SuperApp Server ????????? ???????????? ????????? ?????? ?????? ?????? WAPL TF??? ?????? ?????????????????? ?????? ??????????????? ???????????????.

// ???????????????.


// - ????????? ??????.


// ??? SuperApp Server Runtime :
// http://192.168.9.12/binary/super-app-runtime/super-app-runtime-${version}

// ??? SuperApp Server Maven Repository :
// http://192.168.9.12:8081/#browse/browse:maven-releases:com%2Ftmax%2Fsuper-app-server%2F0.0.5%2Fsuper-app-server-${version}.jar

// ??? SuperApp Server Project :
// http://gitlab.ck:10081/superobject/super-object/tree/release-${version}

// ??? gitlab.ck:10081 ?????? ?????? :
// Default DNS 192.168.1.150 ??? ??????

// """,
//                         to: "dohyun_kim5@tmax.co.kr; ck1@tmax.co.kr; ck2@tmax.co.kr; cqa1@tmax.co.kr;",
//                         from: "dohyun_kim5@tmax.co.kr"
//                 )
//             }
//         }
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