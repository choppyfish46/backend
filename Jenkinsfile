#!groovy
@Library('ddo-pipeline-library') _

// Config =========================
slackURL = 'https://hooks.slack.com/services/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'
slackNotificationChannel = "ow-devops-notifications"
//branch = "${env.BRANCH_NAME}"
branch="develop"

env.MAVEN_PROJECT_BRANCH = "${env.BRANCH_NAME}" // needed to use maven-git-versioning-extension in headless mode
short_branch = "${env.BRANCH_NAME}".replaceAll("/","-")
serviceName = 'navify-ow-bootstrap'

awsCredentialsId = 'ow-deployer.creds'
awsRegion = 'us-west-2'

ecrUrl = 'https://121749245160.dkr.ecr.us-west-2.amazonaws.com'
owecrUrl = 'https://923449152633.dkr.ecr.us-west-2.amazonaws.com'


service = serviceName + '-microservice'
docker_image_name = serviceName
echo "docker_image_name is ${docker_image_name}"
develop_branch = "develop" //changed to test if working with Master
release_branch = "release"
if (branch.startsWith(release_branch)) {
    ddo_branch = "${branch}"
} else {
    ddo_branch = "develop"
}

configFileId = "44ba7186-c2e9-447c-92cb-4e56a1a3d968"

properties([
    disableConcurrentBuilds(),
    buildDiscarder(logRotator(daysToKeepStr: '7', numToKeepStr: '10'))
])

// Stages =========================
node('docker-host') {
    step([$class: 'WsCleanup'])

    timestamps {
        try {
            stage('Checkout') {

                dir(service) {
                    checkout()

                }
            }

            dir(service) {
                docker.image('localstack/localstack:0.8.7').withRun(
                                                            '-e SERVICES=s3:4572,sqs:4576 ' +
                                                            '-e DEFAULT_REGION=us-west-2 ' +
                                                            '-e HOSTNAME_EXTERNAL=s3.localstack.navify ' +
                                                            '--expose 4570-4576 ')
                   {  l ->
                        docker.image(ecrUrl.replace("https://","")+'/ddo-jenkins-slave-centos-saas:latest').inside(
                          "--link ${l.id}:s3.localstack.navify " +
                          '-v /opt/jenkins-slave/m2_repo:/root/.m2') {
                            withEnv([]) {
                                sh 'echo "127.0.0.1 s3.us-east-2.mysite.com" >> /etc/hosts'

                                stage('Unit Tests') {
                                    try {
                                        unitTests();
                                    } finally {
                                        junit '**/surefire-reports/**/*.xml'
                                    }

                                }
                                // this is commented as media needs special ec2 env for
                                // running int tests
                                stage('Integration Tests') {
                                //    integrationTests()
                                }

                                stage('Publish Results') {
                                   // publishResults()
                                }

                                stage('Sonar Analysis') {
                                    //sonarAnalysis()
                                }

                                stage('CodeAnalysis') {
                                    whitesource()
                                }

                                stage('Deploy (to Artifactory)') {
                                    echo 'before should deploy'
                                    if (shouldDeploy()) {
                                        deployToArtifactory()
                                    }
                                }
                            }
                        }

                }

                if (shouldDeploy()) {
                    echo 'in should deploy'
                    image_tag = genImageTag fileId: configFileId

                    stage('Build/Test/Push image to registry') {
                        sh "cp target/${docker_image_name}-${image_tag}.jar infra/docker/" //change file from war to jar
                        dir('infra/docker') {
                            imageToRegistry(image_tag)
                        }
                    }
                }
            }
        } catch (Exception e) {
            println("Caught exception: " + e)
            error = catchException exception: e
        } finally {
            println("CurrentBuild result: " + currentBuild.result)
            sendSlack()
        }
    }
}

boolean shouldDeploy() {
    echo 'should deploy boolean'
    print develop_branch
    print branch
    return branch == develop_branch //|| branch.startsWith(release_branch) || branch.endsWith("-deployable")
}

 def unitTests(boolean skip = false) {
    if (skip) {
        echo "skip unit tests"
    } else {
        mavenHelper fileId: configFileId, options: 'clean',
            extraParams: '-Borg.jacoco:jacoco-maven-plugin:prepare-agent test -P jenkins'
    }
}

/*

def integrationTests(boolean skip = false) {
    if (skip) {
        echo "skip integration tests"
    } else {
        mavenHelper fileId: configFileId, options: 'integration-test verify',
            extraParams: '-Dspring.profiles.active=no-sec'
    }
}

def publishResults(boolean skip = false) {
    if (skip) {
        echo "skip publish results"
    } else {
        publishResults skipArchive: true
    }
}
*/
def sonarAnalysis(boolean skip = false) {
    if (skip) {
        echo "Skip Sonar Analysis."
        return
    }
    sonarBranchParams = '-Dsonar.branch.name=' + branch
    echo 'printing sonarBranchParams'
    println sonarBranchParams
    /* if (branch != develop_branch && !branch.startsWith(release_branch)) {
        sonarBranchParams = sonarBranchParams + ' -Dsonar.branch.target=' + develop_branch

    } */
    checkUrlStatus url: "http://sonar.intranet.roche.com/batch/index"
    withSonarQubeEnv('sonar-roche') {
        mavenHelper fileId: configFileId, options: 'sonar:sonar ' + sonarBranchParams
    }
}

def whitesource(boolean skip = false) {
    if (skip) {
        echo "skip whitesource"
    } else {
        if (branch == develop_branch || branch.startsWith(release_branch)) {
            mavenHelper fileId: configFileId, options: 'whitesource:update'
        } else {
            mavenHelper fileId: configFileId, options: 'whitesource:checkPolicies'
        }
    }
}



def deployToArtifactory(boolean skip = false) {
    if (skip) {
        echo "skip deploy to artifactory"
    } else {
        mavenHelper fileId: configFileId, options: '-DskipTests', extraParams: '-B source:jar deploy'
    }
}

def imageToRegistry(image_tag) {
    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: awsCredentialsId, usernameVariable: 'ACCESS_KEY', passwordVariable: 'SECRET_KEY']]) {

        sh(script: "AWS_ACCESS_KEY_ID=${ACCESS_KEY} AWS_SECRET_ACCESS_KEY=${SECRET_KEY} AWS_DEFAULT_REGION=${awsRegion} ../pipeline/scripts/create-repository.sh ${docker_image_name} ${awsRegion} ")

        docker.withRegistry(owecrUrl, awsCredentialsId) {
           def login_cmd = sh(script: "AWS_ACCESS_KEY_ID=${ACCESS_KEY} AWS_SECRET_ACCESS_KEY=${SECRET_KEY} AWS_DEFAULT_REGION=${awsRegion} aws ecr get-login --no-include-email", returnStdout: true)
           sh "#!/bin/sh -e\n ${login_cmd}"
           def dockerArgs = "--build-arg jar_file=${docker_image_name}-${image_tag}.jar --no-cache ."
           new_image = docker.build(docker_image_name, dockerArgs)
           sh "docker images ${docker_image_name}"
           new_image.push("latest")
        }
    }
}

def checkout() {
    checkout scm
    echo 'Pulling... ' + env.GIT_BRANCH
    def commit = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
    env.GIT_COMMIT = commit
}


def emailnotify() {
      
            mail to: 'durgesh.manohar@contractors.roche.com, david.becker.db2@contractors.roche.com ,dhwan.shah@contractors.roche.com, diwakar.chapagain@contractors.roche.com, lakshmikiran.jasthi@contractors.roche.com,motumarri.krishna_teja@contractors.roche.com, prateek.patil@contractors.roche.com , raj.kesavan@roche.com, raman.ramanathan@roche.com, sambasivarao.byrapuneni@roche.com, sarvesh.padwal@contractors.roche.com, sudheer.sangaraju@contractors.roche.com, uday_kumar.goshika@contractors.roche.com ', from: 'durgesh.manohar@contractors.roche.com',
            subject: "Build: ${env.JOB_NAME}", 
             body: "Job status - \"${env.JOB_NAME}\" build: ${env.BUILD_NUMBER}\n\nView the log at:\n ${env.BUILD_URL}\n\nBlue Ocean:\n${env.RUN_DISPLAY_URL}"
        }

def sendSlack() {
    sendNotification slackChannel: slackNotificationChannel,
            slackHookUrl: slackURL,
            serviceName: serviceName,
            isSimpleMessage: true
}
