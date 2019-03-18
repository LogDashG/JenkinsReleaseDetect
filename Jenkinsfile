#!/usr/bin/env groovy
/*
 * Jenkinsfile
 * JenkinsHomebrew
 *
 * Checks for new versions of Jenkins, updating the core formula when found.
 */

import jenkins.model.*

def MAIL_TO = JenkinsLocationConfiguration.get().getAdminAddress()
echo "MAIL_TO: $MAIL_TO"

properties([
    // Don't trigger this job when changes are found from branch indexing.
    //overrideIndexTriggers(false),
    disableConcurrentBuilds(),
    pipelineTriggers([
        cron('H 1 * * *')
    ]),
    buildDiscarder(logRotator(numToKeepStr: '100')),
])

// Global variables
String version
String file
String url

try {
    timeout(time: 1, unit: 'HOURS') {
        withEnv(['LANG=en_US.UTF-8']) {
            node {
                stage("✨ Latest Version") {
                    // Parse version from homebrew formula json
                    // https://jenkins.io/doc/pipeline/steps/workflow-durable-task-step/#sh-shell-script
                    version = sh(
                        script: """
                            brew info --json=v1 jenkins \
                                | jq .[0].versions.stable \
                                | tr -d '"'
                        """,
                        label: "✨ Version",
                        returnStdout: true
                    ).trim()
                    echo "Jenkins formula version: $version"
                }
                stage("📡 Check for new") {
                    // Increment minor version
                    def (major, minorString) = version.tokenize('.')
                    Integer minor = minorString as Integer
                    minor++
                    version = "$major.$minor"

                    echo "Checking for new Jenkins release with version: $version"
                    file = "jenkins-${version}.war"
                    url = "http://mirrors.jenkins.io/war/$version/jenkins.war"

                    sh(
                        script: """
                            curl \
                                --head \
                                --fail \
                                --silent \
                                $url
                        """,
                        label: "✔️ Check"
                    )
                }
                stage("👇🏻 Download") {
                    echo "👇🏻 Downloading Jenkins $version - $url"
                    sh(
                        script: """
                            curl \
                                --fail \
                                --location \
                                --silent \
                                --output $file \
                                $url
                        """,
                        label: "👇🏻 Download"
                    )
                    String hash = sh(
                        script: """
                            shasum \
                                --algorithm 256 \
                                $file \
                            | cut -d' ' -f1
                        """,
                        label: "🔢 Hash",
                        returnStdout: true
                    ).trim()
                    echo "File hash: $hash"
                }
                stage("🍼 Formula") {

                }
            }
        }
    }
}
catch (Exception ex) {
    String jobName = env.JOB_NAME
    String buildNumber = env.BUILD_NUMBER

    String subject = "👷🏻‍♂️ [FAILURE] $jobName - Build #$buildNumber"
    String body = """$jobName
                |Build #${buildNumber} - FAILURE
                |
                |${env.BUILD_URL}
                |
                |${ex.message}
                |
                |${env.BUILD_URL}console
                |""".stripMargin()

    mail to: MAIL_TO,
         subject: subject,
         body: body
    throw ex
}
