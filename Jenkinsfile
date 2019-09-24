/*
 * Licensed to the Apache Software Foundation (ASF) under one
 * or more contributor license agreements.  See the NOTICE file
 * distributed with this work for additional information
 * regarding copyright ownership.  The ASF licenses this file
 * to you under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance
 * with the License.  You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */

properties([buildDiscarder(logRotator(artifactNumToKeepStr: '5', numToKeepStr: env.BRANCH_NAME=='master'?'5':'1'))])

def buildOs = 'linux'
def buildJdk = '8'
def buildMvn = '3.6.0'
def runITsOses = ['linux', 'windows']
def runITsJdks = ['7', '8', '11','12']
def runITsMvn = '3.6.0'
def runITscommand = "mvn clean install -Prun-its,embedded -B -U -V" // -DmavenDistro=... -Dmaven.test.failure.ignore=true
def tests

try {

def osNode = jenkinsEnv.labelForOS(buildOs) 
node(jenkinsEnv.nodeSelection(osNode)) {
    dir('build') {
        stage('Checkout') {
            checkout scm
        }

        def WORK_DIR=pwd()
        def MAVEN_GOAL='verify'

        stage('Configure deploy') {
           if (env.BRANCH_NAME == 'master'){
               MAVEN_GOAL='deploy'
           }
        }

        tests = resolveScm source: [$class: 'GitSCMSource', credentialsId: '', id: '_', remote: 'https://gitbox.apache.org/repos/asf/maven.git', traits: [[$class: 'jenkins.plugins.git.traits.BranchDiscoveryTrait'], [$class: 'GitToolSCMSourceTrait', gitTool: 'Default']]], targets: [BRANCH_NAME, 'master']
    }
}

Map runITsTasks = [:]
for (String os in runITsOses) {
    for (def jdk in runITsJdks) {
        String osLabel = jenkinsEnv.labelForOS(os);
        String jdkName = jenkinsEnv.jdkFromVersion(os, "${jdk}")
        String mvnName = jenkinsEnv.mvnFromVersion(os, "${runITsMvn}")
        echo "OS: ${os} JDK: ${jdk} => Label: ${osLabel} JDK: ${jdkName}"

        def cmd = [
          'mvn', 'clean',
          'verify',
          '-DskipTests', '-Drat.skip'
        ]
        if (jdk == '7') {
          // Java 7u80 has TLS 1.2 disabled by default: need to explicitely enable
          cmd += '-Dhttps.protocols=TLSv1.2'
        }

        String stageId = "${os}-jdk${jdk}"
        String stageLabel = "Rebuild ${os.capitalize()} Java ${jdk}"
        runITsTasks[stageId] = {
            node(jenkinsEnv.nodeSelection(osLabel)) {
                def WORK_DIR=pwd()
                stage("${stageLabel}") {
                    echo "NODE_NAME = ${env.NODE_NAME}"
                    checkout tests
                    withMaven(jdk: jdkName, maven: mvnName, mavenLocalRepo:"${WORK_DIR}/.repository", options:[
                        artifactsPublisher(disabled: false),
                        junitPublisher(ignoreAttachments: false),
                        findbugsPublisher(disabled: false),
                        openTasksPublisher(disabled: false),
                        dependenciesFingerprintPublisher(),
                        invokerPublisher(),
                        pipelineGraphPublisher()
                    ]) {
                      commitId = ''
                      if (isUnix()) {
                        commitId = sh(returnStdout: true, script: 'git rev-parse HEAD')
                      } else {
                        commitId = bat(returnStdout: true, script: 'git rev-parse HEAD')
                      }
                      cmd += "-DbuildId=${os}-jdk${jdk}-${commitId}"
                      if (isUnix()) {
                        sh cmd.join(' ')
                      } else {
                        bat cmd.join(' ')
                      }
                    }
                }
            }
        }
    }
}

// run the parallel ITs
parallel(runITsTasks)

// JENKINS-34376 seems to make it hard to detect the aborted builds
} catch (org.jenkinsci.plugins.workflow.steps.FlowInterruptedException e) {
    echo "[FAILURE-002] FlowInterruptedException ${e}"
    // this ambiguous condition means a user probably aborted
    if (e.causes.size() == 0) {
        currentBuild.result = "ABORTED"
    } else {
        currentBuild.result = "FAILURE"
    }
    throw e
} catch (hudson.AbortException e) {
    echo "[FAILURE-003] AbortException ${e}"
    // this ambiguous condition means during a shell step, user probably aborted
    if (e.getMessage().contains('script returned exit code 143')) {
        currentBuild.result = "ABORTED"
    } else {
        currentBuild.result = "FAILURE"
    }
    throw e
} catch (InterruptedException e) {
    echo "[FAILURE-004] ${e}"
    currentBuild.result = "ABORTED"
    throw e
} catch (Throwable e) {
    echo "[FAILURE-001] ${e}"
    currentBuild.result = "FAILURE"
    throw e
} finally {
    // notify completion
    stage("Notifications") {
        jenkinsNotify()      
    }    
}

def archiveDirs(stageId, archives) {
    archives.each { archivePrefix, pathToContent ->
        if (fileExists(pathToContent)) {
            zip(zipFile: "${archivePrefix}-${stageId}.zip", dir: pathToContent, archive: true)
        }
    }
}
