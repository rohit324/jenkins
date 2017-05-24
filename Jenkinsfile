#!groovy
import groovy.json.JsonSlurper

// Build Setup
properties([
    [$class: 'BuildDiscarderProperty', strategy:
        [$class: 'LogRotator',
         artifactDaysToKeepStr: '5',
         artifactNumToKeepStr: '5',
         daysToKeepStr: '10',
         numToKeepStr: '10'
        ]
    ]
])

// Build Pipeline
node {
    def jenkinsChannel = "jenkins_crowsnest"
    def authorSlack = isMaster() || isDevelop() ? jenkinsChannel : getSlackNameByAuthor(env.CHANGE_AUTHOR)
    def prLabels = getPrLabels(getPrNumber())
    def channels = getChannelsFromLabels(prLabels)

    try {
        def buildLabels = isDevelop() || isMaster() ? [/* use defaults */] : prLabels
        String slackMessage = mainBuild(buildLabels)

        if (!prLabels.contains("jenkins : slack : off")) {
            def warnAboutLabels = hasWarningLabels(prLabels)
            String slackLevel = warnAboutLabels ? "warning" : "good"

            slackMessage = warnAboutLabels ?
                "*WARNING PR-${getPrNumber()} has the following labels*:\n"
                    .plus("```${prLabels.join('\n')}```\n\n")
                    .plus(slackMessage) :
                isPR() ? slackMessage.plus('*PR Labels:*\n').plus("```${prLabels.join("\n")}```") : slackMessage

            for (String slackTo : channels) {
                slack(slackTo, slackLevel, slackMessage)
            }

            if (!channels.contains(authorSlack)) {
                slack(authorSlack, slackLevel, slackMessage)
            }
        }
    } catch (ex) {
        def String errorMessage =
            "FAILURE on commit:\n"
                .plus("```${gitSummary()}```\n")
                .plus("Jenkins error: \n")
                .plus("```${ex}```\n")

            if (getPrNumber() != null) {
                errorMessage = errorMessage.plus("*PR Labels:* \n")
                    .plus("```${prLabels.join("\n")}```\n")
            }

        if (!prLabels.contains("jenkins : slack : off")) {
            for (String slackTo : channels) {
                slack(slackTo, 'danger', errorMessage)
            }

            if (!channels.contains(authorSlack)) {
                slack(authorSlack, 'danger', errorMessage)
            }
        }

        error errorMessage.plus(ex)
        throw ex
    }
}

def mainBuild(labels) {
    echo "Checking out ${env.BRANCH_NAME}"
    checkout scm

    echo "PR labels: \n"
        .plus(labels.join("\n"))

    def s3BucketName = isMaster() ? "crowsnest-prod" : isDevelop() ? "crowsnest-dev" : "crowsnest-${env.BRANCH_NAME.toLowerCase()}"
    final String frontEndUrl = "http://${s3BucketName}.s3-website-us-west-2.amazonaws.com";

    // maven build
    if (!labels.contains("jenkins : skip : mvn")) {
        def skipTests = labels.contains("jenkins : skip : unit-tests") || labels.contains("jenkins : skip : all-tests")
        def synchronizeBuild = labels.contains("jenkins : mvn : synchronize-build")
        doMvn(skipTests, !synchronizeBuild)
    }

    // back end aws deploy
    if (!labels.contains("jenkins : skip : deploy-BE") &&
        !labels.contains("jenkins : skip : deploy")) {
        def regions = getRegionsFromLabels(labels)
        if (regions.size() == 0) {
            regions = ["us-west-2"] // default to oregon
        }

        doDeployServerless(regions[0])

        // TODO can only deploy to one region at a time because we would have naming conflicts with some resources if there were two regions
        /*for (def region : regions) {
            doDeployServerless(region)
        }*/
    }

    // front end aws deploy
    if (!labels.contains("jenkins : skip : deploy-FE") &&
        !labels.contains("jenkins : skip : deploy")) {
        doDeployFrontEnd(s3BucketName)
    }

    // ui selenium tests
    if (!labels.contains("jenkins : skip : ui-tests(selenium)") &&
        !labels.contains("jenkins : skip : all-tests")) {
        def browsers = getBrowsersFromLabels(labels)
        if (browsers.contains("all")) {
            browsers = [ "chrome", "firefox" ]
        }

        if (browsers.size() == 0) {
            browsers = ["chrome"]
        }

        for (def browser : browsers) {
            doSelenium(frontEndUrl, browser)
        }
    }

    // e2e tests
    if (!labels.contains("jenkins : skip : e2e-tests(regatta)") &&
        !labels.contains("jenkins : skip : all-tests")) {
        doE2E()
    }

    // result slack message
    def prNum = getPrNumber();
    def pullRequest = getPullRequest(prNum);
    def title = pullRequest != null ? "*${prNum != null ? 'PR-' + prNum + ": " : ""}${pullRequest.title}* \n" : ""
    def body = pullRequest != null ? "${pullRequest.body} \n" : ""
    def endpoints = fileExists("./build/endpoints.js") ? "```${readFile("./build/endpoints.js")}``` \n" : ''

    return "${title}"
        .plus(body)
        .plus(endpoints != '' ? "*REST urls:* \n" : '')
        .plus(endpoints)
        .plus("*Front end urls:* \n")
        .plus("<${frontEndUrl}/login|Sign in> \n")
        .plus("<${frontEndUrl}/register|Register> \n")
        .plus("<${frontEndUrl}|Default Page (Access Points)> \n")
        .plus("<${frontEndUrl}/access-points|Access Points> \n")
        .plus("<${frontEndUrl}/data-stores|Data Stores> \n")
        .plus("<${frontEndUrl}/data-flows|Data Flows> \n")
        .plus("<${frontEndUrl}/jobs|Jobs> \n")
}

def doMvn(skipTests, parallelBuild) {
    stage "mvn"

    insideMvn {
        sh "mvn clean install \
            ${parallelBuild ? "-T 1C" : ""} \
            -s settings.xml \
            -D maven.test.skip=${skipTests} \
            -D buildHash=${getCommitHash().take(8)} \
            -D settings.security=settings-security.xml"
    }

    if (!skipTests) {
        // Attach coverage results to pipeline view
        publishHTML(
            target: [
                reportDir  : 'server/dataflow/core/target/site/jacoco',
                reportFiles: 'index.html',
                reportName : 'Unit Test Code Coverage for crowsnest-api-core'
            ]
        )
        publishHTML(
            target: [
                reportDir  : 'server/dataflow/rest/target/site/jacoco',
                reportFiles: 'index.html',
                reportName : 'Unit Test Code Coverage for crowsnest-api-rest'
            ]
        )
    }
}

def doDeployServerless(region) {
    // Deploy to AWS
    stage "Deploy Serverless ${region}"

    final String serverlessStage =
        isMaster() ? "prod" : isDevelop() ? "dev" : "${env.BRANCH_NAME}"

    insideServerless {
        sh "./build/deploy.sh ${serverlessStage} ${region}" // deploy API Gateway and Lambdas
    }
}

def doDeployFrontEnd(s3BucketName) {
    stage "Deploy Front End"

    sh "gzip -f < ./build/endpoints.js > ./build/endpoints.js.gz && \
        mv ./build/endpoints.js.gz ./client/dist/endpoints.js" // inject rest endpoints into FE

    // deploy the FE to S3
    insideAwsCli {
        if (!doesS3BucketExist(s3BucketName)) {
            echo "bucket ${s3BucketName} does not exist. Creating bucket now..."
            sh "aws s3 mb s3://${s3BucketName}" // make bucket
        }
        sh "aws s3 sync ./client/dist s3://${s3BucketName} --acl \"public-read\" --content-encoding \"gzip\""
        sh "aws s3 website s3://${s3BucketName} --index-document index.html --error-document index.html" // host the bucket as a web site
    }
}

def doSelenium(frontEndUrl, browser) {
    stage "Selenium ${browser}"

    withMailosaurCreds { mailApiKey ->
        withEnv(["MAILOSAUR_APIKEY=${mailApiKey}"]) {
            runSeleniumUiTests("${env.BRANCH_NAME}-${env.BUILD_NUMBER}", frontEndUrl, browser);
        }
    }
}

def doE2E() {
    stage "E2E Tests"

    echo "Running E2E tests for crowsnest branch ${env.BRANCH_NAME} and boomvang branch develop"
    build job: '/Regatta-Pipeline/develop', parameters: [
        [$class: 'StringParameterValue',
         name  : 'AGENTS_BUILD',
         value : "develop"
        ],
        [$class: 'StringParameterValue',
         name  : 'SERVER_BUILD',
         value : "${env.BRANCH_NAME}"
        ]
    ]
}

private def runSeleniumUiTests(def buildTag, def crowsnestUrl, def browser) {
    def selenRootPath = "${pwd()}/test/ui/docker"
    def composePath = browser.equals("ie") || browser.equals("ie10") || browser.equals("ie11") ? "${selenRootPath}/internet-explorer" : selenRootPath

    try {
        sh "${composePath}/run-jenkins.sh $browser $crowsnestUrl $buildTag"
    } finally {
        sh "${composePath}/down.sh $browser $buildTag"
    }
}

def insideMvn(Closure closure) {
    sh "docker build -t crowsnest-maven:0.0.1 -f ./build/mvn-Dockerfile ./build"
    def mvnImage = docker.image("crowsnest-maven:0.0.1")

    withStormpathCreds { spHref, spApiKey, spApiKeySecret ->
        withMailosaurCreds { mailApiKey ->
            mvnImage.inside(
                "-e STORMPATH_APPLICATION_HREF=${spHref} \
                 -e STORMPATH_CLIENT_APIKEY_ID=${spApiKey} \
                 -e STORMPATH_CLIENT_APIKEY_SECRET=${spApiKeySecret} \
                 -e MAILOSAUR_APIKEY=${mailApiKey} \
                 -e BRANCH_NAME=${env.BRANCH_NAME}"
            ) {
                closure()
            }
        }
    }

}

def insideServerless(Closure closure) {
    final slsVersion = "1.10.0"

    sh "docker build -t serverless:${slsVersion} --build-arg SERVERLESS_VERSION=${slsVersion} -f build/serverless-Dockerfile ./build"

    def slsImage = docker.image("serverless:${slsVersion}")
    withStormpathCreds { spHref, spApiKey, spApiKeySecret ->
        withAwsCreds { awsUser, awsPass ->
            slsImage.inside(
                "-e AWS_ACCESS_KEY_ID=${awsUser} \
                 -e AWS_SECRET_ACCESS_KEY=${awsPass} \
                 -e AWS_DEFAULT_REGION=us-west-2 \
                 -e AWS_CLIENT_TIMEOUT=60000000 \
                 -e STORMPATH_APPLICATION_HREF=${spHref} \
                 -e AWS_CLIENT_TIMEOUT=60000000 \
                 -e STORMPATH_CLIENT_APIKEY_ID=${spApiKey} \
                 -e STORMPATH_CLIENT_APIKEY_SECRET=${spApiKeySecret}"
            ) {
                closure()
            }
        }
    }
}

def insideAwsCli(Closure closure) {
    sh "docker build -t awscli:latest -f build/awscli-Dockerfile ./build"

    def awsCliImage = docker.image("awscli:latest")

    withAwsCreds { awsUser, awsPass ->
        awsCliImage.inside(
            "-e AWS_ACCESS_KEY_ID=${awsUser} \
             -e AWS_SECRET_ACCESS_KEY=${awsPass} \
             -e AWS_DEFAULT_REGION=us-west-2"
        ) {
            closure()
        }
    }
}

def doesS3BucketExist(String bucketName) {
    try {
        sh "aws s3 ls s3://${bucketName} 2>&1 | tee ${bucketName}-listing.txt"
    } catch (e) {
        println (e)
    }
    return !readFile("${bucketName}-listing.txt").trim().contains("NoSuchBucket")
}

def withMailosaurCreds(Closure closure) {
    return withCredentials([
        [$class: 'UsernamePasswordMultiBinding',
         credentialsId: 'mailosaur',
         usernameVariable: 'MAILOSAUR_USER',
         passwordVariable: 'MAILOSAUR_PASS']
    ]) {
        return closure(env.MAILOSAUR_PASS)
    }
}

def withStormpathCreds(Closure closure) {
    return withCredentials([
        [$class: 'UsernamePasswordMultiBinding',
         credentialsId: 'stormpath',
         usernameVariable: 'APIKEY',
         passwordVariable: 'APIKEY_SECRET']
    ]) {
        def stormpathAppHref = "3bDKC5xMeGzcHfYVOlIbug"
        def STORMPATH_APPLICATION_HREF="https://api.stormpath.com/v1/applications/${stormpathAppHref}"
        return closure(STORMPATH_APPLICATION_HREF, env.APIKEY, env.APIKEY_SECRET)
    }
}

def withDockerCreds(Closure closure) {
    return withCredentials([
        [$class: 'UsernamePasswordMultiBinding',
         credentialsId: 'docker-hub',
         usernameVariable: 'DOCKER_USER',
         passwordVariable: 'DOCKER_PASS']
    ]) {
        return closure(env.DOCKER_USER, env.DOCKER_PASS)
    }
}

def withAwsCreds(Closure closure) {
    return withCredentials([
        [$class: 'UsernamePasswordMultiBinding',
         credentialsId: 'aws-beanstalk',
         usernameVariable: 'AWS_USER',
         passwordVariable: 'AWS_PASS']
    ]) {
        return closure(env.AWS_USER, env.AWS_PASS)
    }
}

def withGithubApiCreds(Closure closure) {
    return withCredentials([
        [$class: 'UsernamePasswordMultiBinding',
         credentialsId: 'github-api',
         usernameVariable: 'GITHUB_BASIC_USER',
         passwordVariable: 'GITHUB_BASIC_TOKEN']
    ]) {
        return closure(env.GITHUB_BASIC_TOKEN)
    }
}

static String getSlackNameByAuthor(def githubId) {
    final static slackByAuthor = [
        'rkrier85233': '@rkrier',
        'Jeff-Cortese': '@jeff',
        'chadgooch': '@chad',
        'burchtimp': '@burchtimp',
        'w00fel': '@sbiedlingmaier',
        'abortman': '@aaron',
        'ggovindaraj': '@ggovindaraj',
        'StanFisher': '@stan.fisher'
    ]

    def name = slackByAuthor.get(githubId)

    return name != null ? name : 'jenkins_crowsnest'
}

def getChannelsFromLabels(labels) {
    def channels = []
    def slackLabelPrefix = "jenkins : slack : "

    for (def label : labels) {
        if (label.startsWith(slackLabelPrefix)) {
            channels.add(label.substring(slackLabelPrefix.length()))
        }
    }

    return channels
}

def getRegionsFromLabels(labels) {
    def regions = []
    def regionLabelPrefix = "jenkins : aws : region : "

    for (def label : labels) {
        if (label.startsWith(regionLabelPrefix)) {
            regions.add(label.substring(regionLabelPrefix.length()))
        }
    }

    return regions
}

def hasWarningLabels(labels) {
    for (def label : labels) {
        if (label.startsWith("jenkins : skip : ")) {
            return true;
        }
    }

    return false
}

def getBrowsersFromLabels(labels) {
    def browsers = []
    def browsersLabelPrefix = "jenkins : test : ui-"

    for (def label : labels) {
        if (label.startsWith(browsersLabelPrefix)) {
            browsers.add(label.substring(browsersLabelPrefix.length()))
        }
    }

    return browsers
}

String getCommitHash() {
    sh 'git rev-parse HEAD > commit.txt'
    return readFile('commit.txt').trim()
}

String getShortCommitHash() {
    return getCommitHash().take(7)
}

String getActualBranchName() {
    sh 'git name-rev --name-only HEAD > branchname'
    return readFile('branchname').trim().replaceAll('remotes/origin/', '')
}

boolean isDevelop() {
    return "${env.BRANCH_NAME}" == "develop"
}

boolean isMaster() {
    return "${env.BRANCH_NAME}" == "master"
}

boolean isPR() {
    return "${env.BRANCH_NAME}".startsWith('PR-')
}

def slack(String channel, String color, String message) {
    echo "sending slack message to ${channel} with status of ${color}: \n ${message}"
    message = "<${env.BUILD_URL}|*${env.JOB_NAME} - Build ${env.BUILD_NUMBER}*>\n"
        .plus("${message}")

    // Don't slack from a Docker user
    if (env.DOCKER_HOST_IP == null) {
        slackSend channel: "${channel}", color: "${color}", message: "${message}"
    }
}

private def gitSummary() {
    sh 'git show --summary > git-summary.txt'
    readFile 'git-summary.txt'
}

def listEnv() {
    sh 'env > env.txt'
    readFile('env.txt').split("\r?\n").each {
        println it
    }
}

def getPullRequest(prNumber) {
    if (prNumber == null) { return null; }
    try {
        return withGithubApiCreds { token ->
            def prResponse = httpRequest customHeaders: [[name: 'Authorization', value: "Basic ${token}"]], url: "https://api.github.com/repos/CleoDev/crowsnest/pulls/${prNumber}"
            return new JsonSlurper().parseText(prResponse.content.toString())
        }
    } catch (ex) {
        println("Unable to retrieve PR ${prNumber}: ${ex}");
    }

    return null;
}

def getPrLabels(prNumber) {
    try {
        return withGithubApiCreds { token ->
            def issueResponse = httpRequest customHeaders: [[name: 'Authorization', value: "Basic ${token}"]], url: "https://api.github.com/repos/CleoDev/crowsnest/issues/${prNumber}"
            def issue = new JsonSlurper().parseText(issueResponse.content.toString())
            labels = []
            for (def label : issue.labels) {
                labels.add(label.name)
            }
            return labels
        }
    } catch (ex) {
        println("Unable to retrieve PR ${prNumber} labels ${ex}")
    }

    return [];
}

def getPrNumber() {
    try {
        String summary = gitSummary();

        if (isPR()) {
            return env.BRANCH_NAME.substring(3);
        } else if (summary.contains("Merge pull request #")) {
            String beginning = summary.substring(summary.indexOf("#") + 1)

            // ugh, String.takeUntil returns a boolean in jenkins for some reason...
            def endIndex = beginning.indexOf(" ")
            if (endIndex >= 1) {
                return beginning.substring(0, endIndex)
            }

            return null
        }
    } catch (ex) {
        println("Error trying to get the PR number: ${ex}")
    }

    return null
}

