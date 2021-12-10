pipeline {
    agent any
    tools {
        gradle 'Gradle 7'
        jdk 'Java 8'
    }
    options {
        buildDiscarder(logRotator(artifactNumToKeepStr: '5'))
    }
    stages {
        stage ('Build') {
            steps {
                sh 'git submodule update --init --recursive'
                sh './gradlew clean build'
            }
            post {
                success {
                    archiveArtifacts artifacts: '**/build/libs/floodgate-*.jar', excludes: "**/build/libs/floodgate-api.jar,**/build/libs/floodgate-core.jar", fingerprint: true
                }
            }
        }

        stage ('Deploy') {
            when {
                anyOf {
                    branch "master"
                    branch "dev/2.1.1"
                }
            }

            steps {
                rtGradleDeployer(
                        id: "GRADLE_DEPLOYER",
                        serverId: "opencollab-artifactory",
                        repo: "opencollab-deployer",
                        releaseRepo: "maven-releases",
                        snapshotRepo: "maven-snapshots"
                )
                rtGradleResolver(
                        id: "GRADLE_RESOLVER",
                        serverId: "opencollab-artifactory"
                )
                rtGradleRun(
                        usesPlugin: true,
                        tool: 'Gradle 7',
                        rootDir: "",
                        useWrapper: true,
                        buildFile: 'build.gradle.kts',
                        tasks: 'build artifactoryPublish',
                        deployerId: "GRADLE_DEPLOYER",
                        resolverId: "GRADLE_RESOLVER"
                )
                rtPublishBuildInfo(
                        serverId: "opencollab-artifactory"
                )
            }
        }
    }

    post {
        always {
            script {
                def changeLogSets = currentBuild.changeSets
                def message = "**Changes:**"

                if (changeLogSets.size() == 0) {
                    message += "\n*No changes.*"
                } else {
                    def repositoryUrl = scm.userRemoteConfigs[0].url.replace(".git", "")
                    def count = 0;
                    def extra = 0;
                    for (int i = 0; i < changeLogSets.size(); i++) {
                        def entries = changeLogSets[i].items
                        for (int j = 0; j < entries.length; j++) {
                            if (count <= 10) {
                                def entry = entries[j]
                                def commitId = entry.commitId.substring(0, 6)
                                message += "\n   - [`${commitId}`](${repositoryUrl}/commit/${entry.commitId}) ${entry.msg}"
                                count++
                            } else {
                                extra++;
                            }
                        }
                    }

                    if (extra != 0) {
                        message += "\n   - ${extra} more commits"
                    }
                }

                env.changes = message
            }
            deleteDir()
            withCredentials([string(credentialsId: 'geyser-discord-webhook', variable: 'DISCORD_WEBHOOK')]) {
                discordSend description: "**Build:** [${currentBuild.id}](${env.BUILD_URL})\n**Status:** [${currentBuild.currentResult}](${env.BUILD_URL})\n${changes}\n\n[**Artifacts on Jenkins**](https://ci.opencollab.dev/job/GeyserMC/job/Floodgate)", footer: 'Open Collaboration Jenkins', link: env.BUILD_URL, successful: currentBuild.resultIsBetterOrEqualTo('SUCCESS'), title: "${env.JOB_NAME} #${currentBuild.id}", webhookURL: DISCORD_WEBHOOK
            }
        }
        success {
            script {
                if (env.BRANCH_NAME == 'master') {
                    build propagate: false, wait: false, job: 'GeyserMC/Floodgate-Fabric/master', parameters: [booleanParam(name: 'SKIP_DISCORD', value: true)]
                }
            }
        }
    }
}