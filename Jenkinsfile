pipeline {
  agent any
  // Configuraiton for the variables used for this specific repo
  environment {
    EXT_RELEASE_TYPE = 'github_stable'
    EXT_USER = 'airsonic'
    EXT_REPO = 'airsonic'
    BUILD_VERSION_ARG = 'AIRSONIC_TAG'
    LS_USER = 'linuxserver'
    LS_REPO = 'pipeline-test'
    DOCKERHUB_IMAGE = 'lspipelive/airsonic'
    DEV_DOCKERHUB_IMAGE = 'lspipetest/airsonic'
    PR_DOCKERHUB_IMAGE = 'lspipepr/airsonic'
    BUILDS_DISCORD = credentials('build_webhook_url')
    GITHUB_TOKEN = credentials('github_token')
    DIST_IMAGE = 'alpine'
    DIST_TAG = '3.7'
    DIST_PACKAGES = 'ffmpeg \
                     flac \
                     lame \
                     openjdk8-jre \
                     ttf-dejavu'
  }
  stages {
    // Setup all the basic environment variables needed for the build
    stage("Set ENV Variables base"){
      steps{
        script{
          env.LS_RELEASE = sh(
            script: '''curl -s https://api.github.com/repos/${LS_USER}/${LS_REPO}/releases/latest | jq -r '. | .tag_name' ''',
            returnStdout: true).trim()
          env.LS_RELEASE_NOTES = sh(
            script: '''git log -1 --pretty=%B | sed -E ':a;N;$!ba;s/\\r{0,1}\\n/\\\\n/g' ''',
            returnStdout: true).trim()
          env.GITHUB_DATE = sh(
            script: '''date '+%Y-%m-%dT%H:%M:%S%:z' ''',
            returnStdout: true).trim()
          env.COMMIT_SHA = sh(
            script: '''git rev-parse HEAD''',
            returnStdout: true).trim()
          env.CODE_URL = sh(
            script: '''echo https://github.com/${LS_USER}/${LS_REPO}/commit/${GIT_COMMIT}''',
            returnStdout: true).trim()
          env.DOCKERHUB_LINK = sh(
            script: '''echo https://hub.docker.com/r/${DOCKERHUB_IMAGE}/tags/''',
            returnStdout: true).trim()
          env.PULL_REQUEST = env.CHANGE_ID
        }
        script{
          env.LS_RELEASE_NUMBER = sh(
            script: '''echo ${LS_RELEASE} |sed 's/^.*-ls//g' ''',
            returnStdout: true).trim()
        }
        script{
          env.LS_TAG_NUMBER = sh(
            script: '''#! /bin/bash
                       # Get the commit for the current tag
                       tagsha=$(git rev-list -n 1 ${LS_RELEASE} 2>/dev/null)
                       # If this is a new commit then increment the LinuxServer release version
                       if [ "${tagsha}" == "${COMMIT_SHA}" ]; then 
                         echo ${LS_RELEASE_NUMBER} 
                       else 
                         echo $((${LS_RELEASE_NUMBER} + 1)) 
                       fi''',
            returnStdout: true).trim()
        }
      }
    }
    /* #######################
       Package Version Tagging
       ####################### */
    // If this is an alpine base image determine the base package tag to use
    stage("Set Package tag Alpine"){
      when {
        expression {
          env.DIST_IMAGE = 'alpine'
        }
      }
      steps{
        echo 'Grabbing the latest alpine base image'
        sh '''docker pull alpine:${DIST_TAG}'''
        echo 'Generating the package hash from the current versions'
        script{
          env.PACKAGE_TAG = sh(
            script: '''docker run -it alpine:${DIST_TAG} sh -c 'apk update --quiet && apk info ${DIST_PACKAGES} | md5sum | cut -c1-8' ''',
            returnStdout: true).trim()
        }
      }
    }
    /* ########################
       External Release Tagging
       ######################## */
    // If this is a stable github release use the latest endpoint from github to determine the ext tag
    stage("Set ENV github_stable"){
      when {
        expression {
          env.EXT_RELEASE_TYPE = 'github_stable'
        }
      }
      steps{
        script{
          env.EXT_RELEASE = sh(
            script: '''curl -s https://api.github.com/repos/${EXT_USER}/${EXT_REPO}/releases/latest | jq -r '. | .tag_name' ''',
            returnStdout: true).trim()
        }
      }
    }
    /* ###############
       Build Container
       ############### */
    // Build Docker container for push to LS Repo
    stage('Build') {
      steps {
          echo "Building most current release of ${EXT_REPO}"
          sh "docker build --no-cache -t ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-ls${LS_TAG_NUMBER} --build-arg ${BUILD_VERSION_ARG}=${EXT_RELEASE} --build-arg VERSION=${LS_TAG_NUMBER} --build-arg BUILD_DATE=${GITHUB_DATE} ."
        }
    }
    /* #######
       Testing
       ####### */
    // Run Container tests
    stage('Test') {
      steps {
       echo 'CI Tests for future use'
      }
    }
    /* ##################
       Live Release Logic
       ################## */
    // If this is a public release push this to the live repo triggered by an external repo update or LS repo update on master
    stage('Docker-Push-Release') {
      when {
        branch "master"
        expression {
          env.LS_RELEASE != env.EXT_RELEASE + '-ls' + env.LS_TAG_NUMBER
        }
      }
      steps {
        withCredentials([
          [
            $class: 'UsernamePasswordMultiBinding',
            credentialsId: 'c1701109-4bdc-4a9c-b3ea-480bec9a2ca6',
            usernameVariable: 'DOCKERUSER',
            passwordVariable: 'DOCKERPASS'
          ]
        ]) {
          echo 'Logging into DockerHub'
          sh '''#! /bin/bash
             echo $DOCKERPASS | docker login -u $DOCKERUSER --password-stdin
             '''
          echo 'First push the latest tag'
          sh "docker tag ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-ls${LS_TAG_NUMBER} ${DOCKERHUB_IMAGE}:latest"
          sh "docker push ${DOCKERHUB_IMAGE}:latest"
          echo 'Pushing by release tag'
          sh "docker push ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-ls${LS_TAG_NUMBER}"
        }
      }
    }
    // If this is a public release tag it in the LS Github and push a changelog from external repo and our internal one
    stage('Github-Tag-Push-Release') {
      when { 
        branch "master"
        expression {
          env.LS_RELEASE != env.EXT_RELEASE + '-ls' + env.LS_TAG_NUMBER
        }
        environment name: 'CHANGE_ID', value: ''
      }
      steps {
        echo "Pushing New tag for current commit ${EXT_RELEASE}-ls${LS_TAG_NUMBER}"
        sh '''curl -H "Authorization: token ${GITHUB_TOKEN}" -X POST https://api.github.com/repos/${LS_USER}/${LS_REPO}/git/tags -d '{"tag":"'${EXT_RELEASE}'-ls'${LS_TAG_NUMBER}'","object": "'${COMMIT_SHA}'","message": "Tagging Release '${EXT_RELEASE}'-ls'${LS_TAG_NUMBER}' to master","type": "commit",  "tagger": {"name": "LinuxServer Jenkins","email": "jenkins@linuxserver.io","date": "'${GITHUB_DATE}'"}}' '''
        echo "Pushing New release for Tag"
        sh '''#! /bin/bash
              # Grabbing the current release body from external repo
              curl -s https://api.github.com/repos/${EXT_USER}/${EXT_REPO}/releases/latest | jq '. |.body' | sed 's:^.\\(.*\\).$:\\1:' > releasebody.json
              # Creating the start of the json payload
              echo '{"tag_name":"'${EXT_RELEASE}'-pkg-'${PACKAGE_TAG}'-ls'${LS_TAG_NUMBER}'","target_commitish": "master","name": "'${EXT_RELEASE}'-pkg-'${PACKAGE_TAG}'-ls'${LS_TAG_NUMBER}'","body": "**LinuxServer Changes:**\\n\\n'${LS_RELEASE_NOTES}'\\n**'${EXT_REPO}' Changes:**\\n\\n' > start
              # Add the end of the payload to the file
              printf '","draft": false,"prerelease": false}' >> releasebody.json
              # Combine the start and ending string This is needed do to incompatibility with JSON and Bash escape strings
              paste -d'\\0' start releasebody.json > releasebody.json.done
              # Send payload to github
              curl -H "Authorization: token ${GITHUB_TOKEN}" -X POST https://api.github.com/repos/${LS_USER}/${LS_REPO}/releases -d @releasebody.json.done'''
      }
    }
    // Use helper container to push the current README in master to the DockerHub Repo
    stage('Sync-README') {
      when {
        branch "master"
        expression {
          env.LS_RELEASE != env.EXT_RELEASE + '-ls' + env.LS_TAG_NUMBER
        }
      }
      steps {
        withCredentials([
          [
            $class: 'UsernamePasswordMultiBinding',
            credentialsId: 'c1701109-4bdc-4a9c-b3ea-480bec9a2ca6',
            usernameVariable: 'DOCKERUSER',
            passwordVariable: 'DOCKERPASS'
          ]
        ]) {
          echo 'Run Docker README Sync'
          sh '''#! /bin/bash
                docker pull lsiodev/readme-sync
                docker run --rm=true \
                  -e DOCKERHUB_USERNAME=$DOCKERUSER \
                  -e DOCKERHUB_PASSWORD=$DOCKERPASS \
                  -e GIT_REPOSITORY=${LS_USER}/${LS_REPO} \
                  -e DOCKER_REPOSITORY=${DOCKERHUB_IMAGE} \
                  -e GIT_BRANCH=master \
                  lsiodev/readme-sync bash -c 'node sync'
             '''
        }
      }
    }
    /* #################
       Dev Release Logic
       ################# */
    // Push to the Dev user dockerhub endpoint when this is a non master branch
    stage('Docker-Push-Dev') {
      when { 
        not { 
         branch "master"
        }
        environment name: 'CHANGE_ID', value: ''
      }
      steps {
        withCredentials([
          [
            $class: 'UsernamePasswordMultiBinding',
            credentialsId: 'c1701109-4bdc-4a9c-b3ea-480bec9a2ca6',
            usernameVariable: 'DOCKERUSER',
            passwordVariable: 'DOCKERPASS'
          ]
        ]) {
          echo 'Logging into DockerHub'
          sh '''#! /bin/bash
             echo $DOCKERPASS | docker login -u $DOCKERUSER --password-stdin
             '''
          echo 'Tag images to the built one'
          sh "docker tag ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-ls${LS_TAG_NUMBER} ${DEV_DOCKERHUB_IMAGE}:latest"
          sh "docker tag ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-ls${LS_TAG_NUMBER} ${DEV_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA}"
          echo 'Pushing both tags'
          sh "docker push ${DEV_DOCKERHUB_IMAGE}:latest"
          sh "docker push ${DEV_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-dev-${COMMIT_SHA}"
        }
        script{
          env.DOCKERHUB_LINK = sh(
            script: '''echo https://hub.docker.com/r/${DEV_DOCKERHUB_IMAGE}/tags/''',
            returnStdout: true).trim()
        }
      }
    }
    /* ################
       PR Release Logic
       ################ */
    // Push to PR user dockerhub endpoint when this is a pull request
    stage('Docker-Push-PR') {
      when {
        not {
          environment name: 'CHANGE_ID', value: ''
        }
      }
      steps {
        withCredentials([
          [
            $class: 'UsernamePasswordMultiBinding',
            credentialsId: 'c1701109-4bdc-4a9c-b3ea-480bec9a2ca6',
            usernameVariable: 'DOCKERUSER',
            passwordVariable: 'DOCKERPASS'
          ]
        ]) {
          echo 'Logging into DockerHub'
          sh '''#! /bin/bash
             echo $DOCKERPASS | docker login -u $DOCKERUSER --password-stdin
             '''
          echo 'Tag images to the built one'
          sh "docker tag ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-ls${LS_TAG_NUMBER} ${PR_DOCKERHUB_IMAGE}:latest"
          sh "docker tag ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-ls${LS_TAG_NUMBER} ${PR_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST}"
          echo 'Pushing both tags'
          sh "docker push ${PR_DOCKERHUB_IMAGE}:latest"
          sh "docker push ${PR_DOCKERHUB_IMAGE}:${EXT_RELEASE}-pkg-${PACKAGE_TAG}-pr-${PULL_REQUEST}"
        }
        script{
          env.CODE_URL = sh(
            script: '''echo https://github.com/${LS_USER}/${LS_REPO}/pull/${PULL_REQUEST}''',
            returnStdout: true).trim()
          env.DOCKERHUB_LINK = sh(
            script: '''echo https://hub.docker.com/r/${PR_DOCKERHUB_IMAGE}/tags/''',
            returnStdout: true).trim()
        }
      }
    }
  }
  /* ######################
     Send status to Discord
     ###################### */
  post { 
    success {
      echo "Build good send details to discord"
      sh ''' curl -X POST --data '{"avatar_url": "https://wiki.jenkins-ci.org/download/attachments/2916393/headshot.png","embeds": [{"color": 1681177,"description": "**Build:**  '${BUILD_NUMBER}'\\n**Status:**  Success\\n**Job:** '${RUN_DISPLAY_URL}'\\n**Change:** '${CODE_URL}'\\n**External Release:**: https://github.com/'${EXT_USER}'/'${EXT_REPO}'/releases/tag/'${EXT_RELEASE}'\\n**DockerHub:** '${DOCKERHUB_LINK}'\\n"}],"username": "Jenkins"}' ${BUILDS_DISCORD} '''
    }
    failure {
      echo "Build Bad sending details to discord"
      sh ''' curl -X POST --data '{"avatar_url": "https://wiki.jenkins-ci.org/download/attachments/2916393/headshot.png","embeds": [{"color": 16711680,"description": "**Build:**  '${BUILD_NUMBER}'\\n**Status:**  failure\\n**Job:** '${RUN_DISPLAY_URL}'\\n**Change:** '${CODE_URL}'\\n**External Release:**: https://github.com/'${EXT_USER}'/'${EXT_REPO}'/releases/tag/'${EXT_RELEASE}'\\n**DockerHub:** '${DOCKERHUB_LINK}'\\n"}],"username": "Jenkins"}' ${BUILDS_DISCORD} '''
    }
  }
}
