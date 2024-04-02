pipeline {

    agent {
        kubernetes {
            defaultContainer 'maven'
            yamlFile 'agent.yaml'
        }
    }

    environment {
        GPG_SECRET     = credentials('presto-release-gpg-secret')
        GPG_TRUST      = credentials("presto-release-gpg-trust")
        GPG_PASSPHRASE = credentials("presto-release-gpg-passphrase")

        GITHUB_OSS_TOKEN_ID = 'github-token-presto-release-bot'

        SONATYPE_NEXUS_CREDS    = credentials('presto-sonatype-nexus-creds')
        SONATYPE_NEXUS_PASSWORD = "$SONATYPE_NEXUS_CREDS_PSW"
        SONATYPE_NEXUS_USERNAME = "$SONATYPE_NEXUS_CREDS_USR"
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '100'))
        disableConcurrentBuilds()
        disableResume()
        overrideIndexTriggers(false)
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
    }

    parameters {
        booleanParam(name: 'PERFORM_MAVEN_RELEASE',
                     defaultValue: false,
                     description: 'Release and update development version')
    }

    stages {
        stage ('Build and Test') {
            steps {
                sh '''
                    mvn clean install
                '''
            }
        }

        stage ('Release Airlift') {
            when {
                expression { params.PERFORM_MAVEN_RELEASE }
            }

            steps {
                script {
                    env.AIRLIFT_RELEASE_VERSION = sh(
                        script: 'mvn org.apache.maven.plugins:maven-help-plugin:3.2.0:evaluate -Dexpression=project.version -q -ntp -DforceStdout',
                        returnStdout: true).trim()
                }
                echo "Airlift current version ${AIRLIFT_RELEASE_VERSION}"
                sh '''
                    mvn release:prepare -B -DskipTests \
                        -DautoVersionSubmodules=true \
                        -DgenerateBackupPoms=false

                    SCM_TAG=$(cat release.properties | grep scm.tag=)
                    echo "Release Airlift ${SCM_TAG#*=}"
                    mvn release:perform -B -DskipTests --dryRun
                '''
            }
        }

        stage ('Create Release Branch') {
            options {
                retry(5)
            }
            steps {
                withCredentials([
                        usernamePassword(
                            credentialsId: "${GITHUB_TOKEN_ID}",
                            passwordVariable: 'GIT_PASSWORD',
                            usernameVariable: 'GIT_USERNAME')]) {
                    sh '''
                        ORIGIN="https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/prestodb/presto.git"
                        git log -5
                    '''
                }
            }
        }
    }
}
