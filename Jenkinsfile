#!/usr/bin/env groovy

// TODO: FINISH AND TEST ME !!!

// https://jenkins.io/doc/book/pipeline/shared-libraries/
// TODO: move to a Jenkins CC library

// export JENKINS_URL=http://ccbvtauto.eur.ad.sag:8080
// curl -X POST -F "jenkinsfile=<Jenkinsfile" $JENKINS_URL/pipeline-model-converter/validate


def install (command) {
    if (isUnix()) {
        sh "bin/sagccantw $command"
    } else {
        // TODO: implement sagccantw for Windows
        bat "ant $command"
    }
}

def ant (command) {
    if (isUnix()) {
        sh "ant $command"
    } else {
        bat "ant $command"
    }
}

def antcc (command) {
    if (isUnix()) {
        sh "bin/antcc $command"
    } else {
        bat "bin\\antcc $command"
    }
}

def restartVMs(propfile) {
    def props = readProperties file: propfile
    def vms = props['vm.names']?.split(',')
    def vmserver = props['vm.server']
    def vmwait = props['vm.wait']?.toInteger()

    if (!vmserver) {
        error message: "Required vm.server, vm.names properties are not set in ${params.CC_ENV} env.properties file"
    }

    def builders = [:]
    for (x in vms) {
        def vm = x
        builders[vm] = {
            node('master') {
                vSphere buildStep: [$class: 'PowerOff', evenIfSuspended: false, shutdownGracefully: false, vm: vm], serverName: vmserver
                vSphere buildStep: [$class: 'PowerOn', timeoutInSeconds: 180, vm: vm], serverName: vmserver
                sleep vmwait
            }
        }                        
    }
    parallel builders // run in parallel
}

def test(propfile) {
    def props = readProperties file: propfile
    def vms = props['vm.names']?.split(',')
    def vmdomain = props['vm.domain']
    def builders = [:]
    for (x in vms) {
        def label = x + vmdomain // Need to bind the label variable before the closure - can't do 'for (label in labels)'
        builders[label] = {
            node(label) {
                unstash 'scripts'
                antcc '-Daccept.license=true boot'
                antcc 'startcc restartcc'
                antcc 'apply -Dt=tests/test-template.yaml'
                antcc 'ps jobs log logs'
                antcc 'stopcc'
            }
        }                        
    }
    parallel builders // kick off parallel provisioning    
}

pipeline {
    agent {
        label 'master'
    }
    options {
        buildDiscarder(logRotator(numToKeepStr:'10'))
        disableConcurrentBuilds()
    }
    environment {
        CC_INSTALLER_URL = "http://aquarius-bg.eur.ad.sag/cc/installers" // internal download site
        CC_ENV_FILE = "tests/test.properties"
        CC_PASSWORD = "manage"
    }
    stages {
        stage("Prepare") {
            steps {
                stash 'scripts'
            }
        }        
        stage("Unit Test") {
            agent {
                docker { image 'ccdevops/java-build-tools' } // with ant
            }
            steps {
                unstash 'scripts'
                timeout (time:10, unit:'MINUTES') {
                    sh "ant -Dinstall.dir=`pwd`/build/cli client"
                }
            }
            post {
                always {
                    sh "ant -Dinstall.dir=`pwd`/build/cc/cli uninstall clean"
                }
            }
        }    
        stage ('Dockerize') {
            agent {
              label 'docker'
            } 
            steps {
              unstash 'scripts'
              sh 'docker-compose build'
              sh 'docker-compose push'        
            }
        }     
        stage ('Restart VMs') { 
            steps {
                script {
                    restartVMs env.CC_ENV_FILE
                }              
            }
        }  
        stage ('Platform Test') {
            steps {
                script {
                    test env.CC_ENV_FILE
                }         
            }
        }     
    }
}
