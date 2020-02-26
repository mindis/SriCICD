node {
    def GITREPO         = "/var/lib/jenkins/workspace/${env.JOB_NAME}"
    def GITREPOREMOTE   = "https://github.com/stikkireddy/SriCICD.git"
    def GITHUBCREDID    = "gitsynch"
    def CURRENTRELEASE  = "master"
    def DBTOKEN         = "DemoToken"
    def DBURL           = "https://demo.cloud.databricks.com"
    def SCRIPTPATH      = "${GITREPO}/cicd-scripts"
    def NOTEBOOKPATH    = "${GITREPO}/notebooks"
    def LIBRARYPATH     = "${GITREPO}/libraries"
    def BUILDPATH       = "${GITREPO}/Builds/${env.JOB_NAME}-${env.BUILD_NUMBER}"
    def OUTFILEPATH     = "${BUILDPATH}/logs/Output"
    def TESTRESULTPATH  = "${BUILDPATH}/logs/reports/junit"
    def WORKSPACEPATH   = "<Workspace Dir path e.g. /Shared/ETL>"
    def DBFSPATH        = "dbfs:/Libs/python/"
    def CLUSTERID       = "0226-121723-stud25"
    def CONDAPATH       = "/Users/Sri.Tikkireddy/PycharmProjects/SriCICD/venv"
    def CONDAENV        = "cicddemo"
    def SLACKURL        = "https://hooks.slack.com/services/ABC123/DEF456/DEMO1122"
    def SLACKCHANNEL    = "#demo-notifications"

    stage('Setup') {
        withCredentials([string(credentialsId: DBTOKEN, variable: 'TOKEN')]) {
            withPythonEnv('/Users/Sri.Tikkireddy/.pyenv/versions/3.6.8/bin/python3') {
                sh """# Configure Conda Environment for deployment & testing
                      export LC_ALL=en_US.UTF-8
                      export LANG=en_US.UTF-8

                      python --version

                      pip install databricks-connect
                      pip install databricks-cli

                      # Configure Databricks CLI for deployment
                      echo "${DBURL}
                      $TOKEN" | databricks configure --token

                      # Configure Databricks Connect for testing
                      echo "${DBURL}
                      $TOKEN
                      ${CLUSTERID}
                      0
                      15001" | databricks-connect configure
                   """
           }
        }
    }
    stage('Checkout') { // for display purposes
        echo "Pulling ${CURRENTRELEASE} Branch from Github"
        git branch: CURRENTRELEASE, credentialsId: GITHUBCREDID, url: GITREPOREMOTE
    }
    stage('Run Unit Tests') {
        try {
        withPythonEnv('/Users/Sri.Tikkireddy/.pyenv/versions/3.6.8/bin/python3') {
            sh """
                  python3 -m pip install pytest
                  # Python Tests for Libs
                  python3 -m pytest --junit-xml=${TESTRESULTPATH}/TEST-libout.xml ${LIBRARYPATH}/python/dbxdemo/test*.py || true
               """
       }
        } catch(err) {
          step([$class: 'JUnitResultArchiver', testResults: '--junit-xml=${TESTRESULTPATH}/TEST-*.xml'])
          if (currentBuild.result == 'UNSTABLE')
            currentBuild.result = 'FAILURE'
          throw err
        }
    }
    stage('Package') {
    withPythonEnv('/Users/Sri.Tikkireddy/.pyenv/versions/3.6.8/bin/python3') {
        sh """

              # Package Python Library to Wheel
              cd ${LIBRARYPATH}/python/dbxdemo
              python3 setup.py sdist bdist_wheel
           """
    }
    }
    stage('Build Artifact') {
        sh """mkdir -p ${BUILDPATH}/Workspace
              mkdir -p ${BUILDPATH}/Libraries/python
              mkdir -p ${BUILDPATH}/Validation/Output
              #Get Modified Files
              git diff --name-only --diff-filter=AMR HEAD^1 HEAD | xargs -I '{}' cp --parents -r '{}' ${BUILDPATH}

              # Get Packaged Libs
              find ${LIBRARYPATH} -name '*.whl' | xargs -I '{}' cp '{}' ${BUILDPATH}/Libraries/python/

              # Generate Artifact
              tar -czvf Builds/latest_build.tar.gz ${BUILDPATH}
           """
        archiveArtifacts artifacts: 'Builds/latest_build.tar.gz'
    }
    stage('Deploy') {
        sh """#!/bin/zsh
              # Enable Conda Environment for tests
              source ${CONDAPATH}/bin/activate ${CONDAENV}

              # Use Databricks CLI to deploy Notebooks
              databricks workspace import_dir ${BUILDPATH}/Workspace ${WORKSPACEPATH}

              dbfs cp -r ${BUILDPATH}/Libraries/python ${DBFSPATH}
           """
        withCredentials([string(credentialsId: DBTOKEN, variable: 'TOKEN')]) {
           sh """#!/bin/zsh

                 #Get space delimited list of libraries
                 LIBS=\$(find ${BUILDPATH}/Libraries/python/ -name '*.whl' | sed 's#.*/##' | paste -sd " ")

                 #Script to uninstall, reboot if needed & instsall library
                 python3 ${SCRIPTPATH}/installWhlLibrary.py --shard=${DBURL}\
                    --token=$TOKEN\
                    --clusterid=${CLUSTERID}\
                    --libs=\$LIBS\
                    --dbfspath=${DBFSPATH}
              """
        }
    }
    stage('Run Integration Tests') {
        withCredentials([string(credentialsId: DBTOKEN, variable: 'TOKEN')]) {
            sh """python3 ${SCRIPTPATH}/executenotebook.py --shard=${DBURL}\
                            --token=$TOKEN\
                            --clusterid=${CLUSTERID}\
                            --localpath=${NOTEBOOKPATH}/VALIDATION\
                            --workspacepath=${WORKSPACEPATH}/VALIDATION\
                            --outfilepath=${OUTFILEPATH}
               """
        }
        sh """sed -i -e 's #ENV# ${OUTFILEPATH} g' ${SCRIPTPATH}/evaluatenotebookruns.py
              python3 -m pytest --junit-xml=${TESTRESULTPATH}/TEST-notebookout.xml ${SCRIPTPATH}/evaluatenotebookruns.py || true
           """
    }
    stage('Report Test Results') {
        sh """find ${OUTFILEPATH} -name '*.json' -exec gzip --verbose {} \\;
              touch ${TESTRESULTPATH}/TEST-*.xml
           """
        junit "**/reports/junit/*.xml"

    }
    stage('Send Notifications') {
        sh """python3 ${SCRIPTPATH}/notify.py "--slackurl=${SLACKURL}\
                            --message='${env.JOB_NAME}:Build-${env.BUILD_NUMBER}'\
                            --channel=${SLACKCHANNEL}\
                            --outputpath=${OUTFILEPATH}
           """
    }
}