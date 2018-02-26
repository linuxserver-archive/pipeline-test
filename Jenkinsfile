pipeline {
  agent any
  environment {
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
  }
  stages {
    stage("Set ENV Variables"){
      steps{
        script{
          env.EXT_RELEASE = sh(
            script: '''curl -s https://api.github.com/repos/${EXT_USER}/${EXT_REPO}/releases/latest | jq -r '. | .tag_name' ''',
            returnStdout: true).trim()
          env.LS_RELEASE = sh(
            script: '''curl -s https://api.github.com/repos/${LS_USER}/${LS_REPO}/releases/latest | jq -r '. | .tag_name' ''',
            returnStdout: true).trim()
          env.LS_RELEASE_NOTES = sh(
            script: '''git log -1 --pretty=%B | sed -E ':a;N;$!ba;s/\\r{0,1}\\n/\\\\\\\\n/g' ''',
            returnStdout: true).trim()
          env.GITHUB_DATE = sh(
            script: '''date '+%Y-%m-%dT%H:%M:%S%:z' ''',
            returnStdout: true).trim()
          env.COMMIT_SHA = sh(
            script: '''git rev-parse HEAD''',
            returnStdout: true).trim()
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
    stage('Build') {
      steps {
          echo "Building most current release of ${EXT_REPO}"
          sh "docker build --no-cache -t ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER} --build-arg ${BUILD_VERSION_ARG}=${EXT_RELEASE} --build-arg VERSION=${LS_TAG_NUMBER} --build-arg BUILD_DATE=${GITHUB_DATE} ."
        }
    }
    stage('Test') {
      steps {
       echo 'CI Tests for future use'
      }
    }
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
          sh "docker tag ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DOCKERHUB_IMAGE}:latest"
          sh "docker push ${DOCKERHUB_IMAGE}:latest"
          echo 'Pushing by release tag'
          sh "docker push ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER}"
        }
      }
    }
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
              echo '{"tag_name":"'${EXT_RELEASE}'-ls'${LS_TAG_NUMBER}'","target_commitish": "master","name": "'${EXT_RELEASE}'-ls'${LS_TAG_NUMBER}'","body": "**LinuxServer Changes:**\\n\\n'${LS_RELEASE_NOTES}'\\n**'${EXT_REPO}' Changes:**\\n\\n' > start
              # Add the end of the payload to the file
              printf '","draft": false,"prerelease": false}' >> releasebody.json
              # Combine the start and ending string This is needed do to incompatibility with JSON and Bash escape strings
              paste -d'\\0' start releasebody.json > releasebody.json.done
              # Send payload to github
              curl -H "Authorization: token ${GITHUB_TOKEN}" -X POST https://api.github.com/repos/${LS_USER}/${LS_REPO}/releases -d @releasebody.json.done'''
      }
    }
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
          sh "docker tag ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DEV_DOCKERHUB_IMAGE}:latest"
          sh "docker tag ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${DEV_DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${COMMIT_SHA}"
          echo 'Pushing both tags'
          sh "docker push ${DEV_DOCKERHUB_IMAGE}:latest"
          sh "docker push ${DEV_DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${COMMIT_SHA}"
        }
      }
    }
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
          sh "docker tag ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${PR_DOCKERHUB_IMAGE}:latest"
          sh "docker tag ${DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${LS_TAG_NUMBER} ${PR_DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${COMMIT_SHA}"
          echo 'Pushing both tags'
          sh "docker push ${PR_DOCKERHUB_IMAGE}:latest"
          sh "docker push ${PR_DOCKERHUB_IMAGE}:${EXT_RELEASE}-ls${COMMIT_SHA}"
        }
      }
    }
  }
  post { 
    success {
      echo "Build good send details to discord"
      sh ''' curl -X POST --data '{"avatar_url": "https://wiki.jenkins-ci.org/download/attachments/2916393/headshot.png","embeds": [{"color": 1681177,"description": "**Build:**  '${BUILD_NUMBER}'\\n**Status:**  Success\\n**Job:** '${RUN_DISPLAY_URL}'\\n**Commit:** https://github.com/'${LS_USER}'/'${LS_REPO}'/commit/'${GIT_COMMIT}'\\n**External Release:**: https://github.com/'${EXT_USER}'/'${EXT_REPO}'/releases/tag/'${EXT_RELEASE}'\\n"}],"username": "Jenkins"}' ${BUILDS_DISCORD} '''
    }
    failure {
      echo "Build Bad sending details to discord"
      sh ''' curl -X POST --data '{"avatar_url": "https://wiki.jenkins-ci.org/download/attachments/2916393/headshot.png","embeds": [{"color": 16711680,"description": "**Build:**  '${BUILD_NUMBER}'\\n**Status:**  failure\\n**Job:** '${RUN_DISPLAY_URL}'\\n**Commit:** https://github.com/'${LS_USER}'/'${LS_REPO}'/commit/'${GIT_COMMIT}'\\n**External Release:**: https://github.com/'${EXT_USER}'/'${EXT_REPO}'/releases/tag/'${EXT_RELEASE}'\\n"}],"username": "Jenkins"}' ${BUILDS_DISCORD} '''
    }
  }
}
