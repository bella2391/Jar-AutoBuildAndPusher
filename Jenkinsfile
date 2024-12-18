pipeline {
    agent any
    environment {
        USERNAME = 'bella2391'
        REPO_NAME = 'Dependency-Provider'
        BRANCH_NAME = 'master'
        REPO_URL = "https://github.com/${USERNAME}/${REPO_NAME}.git"
        JAR_PATH = 'build/libs/FMC-Dependency-1.0.0.jar'
        TAG_NAME = "${REPO_NAME}-${BUILD_NUMBER}"
        // Caution: Do not set this value too low, as it may cause Jenkins to get stuck in a loop of aborted builds.
        // build intervals (mininutes)
        BUILD_INTERVAL_MINUTES = 5
        AUTO_BUILD_NOTIFICATION = true
        BUILD_RESULT_NOTIFICATION = true
        WEBHOOK_NAME = 'Jenkins'
        WEBHOOK_AVATAR_URL = 'https://licensecounter.jp/devops-hub/assets/images/products/logo-jenkins680_500.png' // 'https://www.spiceworks.co.jp/blog/wp-content/uploads/2014/06/logo.png'
        AUTO_BUILD_THUMBNAIL_URL = 'https://engineering.tomtom.com/assets/images/java/create-java-jar.png'
        BUILD_RESULT_THUMBNAIL_URL = 'https://meister-kentei.jp/magazine/wp-content/uploads/2022/11/github-6980894_1280.png'
        BUILD_FAILED_THUMBNAIL_URL = 'https://slack.engineering/wp-content/uploads/sites/7/2021/05/jenkins-fire.png'
        WEBHOOK_AUTO_BUILD_COLOR = 16705372 // yellow // blue: 5814783
        WEBHOOK_BUILD_RESULT_COLOR = 5763719 // green
        WEBHOOK_BUILD_RESULT_FAILDED_COLOR = 16711680 // red
    }
    stages {
        stage('Check Build Interval') {
            steps {
                script {
                    def previousBuild = currentBuild.previousBuild
                    if (previousBuild != null) {
                        def currentTime = System.currentTimeMillis()
                        def lastBuildTime = previousBuild.getTimeInMillis()
                        def diffMinutes = (currentTime - lastBuildTime) / (1000 * 60)
                        def diffMinutesNum = diffMinutes.toDouble()
                        def intervalNum = BUILD_INTERVAL_MINUTES.toDouble()
                        echo "Condition result: ${diffMinutesNum < intervalNum}"
                        
                        if (diffMinutesNum < intervalNum) {
                            currentBuild.result = 'ABORTED'
                            error("Build aborted: Only ${diffMinutes} minutes since last build. Must wait at least ${BUILD_INTERVAL_MINUTES} minutes.")
                        }
                    }
                }
            }
        }
        stage('Auto Build Notification') {
            steps {
                script {
                    if (AUTO_BUILD_NOTIFICATION) {
                        withCredentials([string(credentialsId: 'Discord-Webhook-URL', variable: 'DISCORD_WEBHOOK_URL')]) {
                            def payload = """
                            {
                                "username": "${WEBHOOK_NAME}",
                                "avatar_url": "${WEBHOOK_AVATAR_URL}",
                                "embeds": [{
                                    "title": "自動ビルド通知",
                                    "description": "**ビルド情報**",
                                    "color": ${WEBHOOK_AUTO_BUILD_COLOR},
                                    "author": {
                                        "name": "${USERNAME}",
                                        "icon_url": "https://github.com/${USERNAME}.png",
                                        "url": "https://github.com/${USERNAME}"
                                        },
                                    "thumbnail": {
                                        "url": "${AUTO_BUILD_THUMBNAIL_URL}"
                                    },
                                    "fields": [
                                        {
                                            "name": "ビルドURL",
                                            "value": "${BUILD_URL}",
                                            "inline": true
                                        },
                                        {
                                            "name": "ビルド対象のレポジトリ",
                                            "value": "${REPO_URL}",
                                            "inline": true
                                        }
                                    ],
                                    "timestamp": "${new Date().format("yyyy-MM-dd'T'HH:mm:ss.SSS'Z'", TimeZone.getTimeZone('Asia/Tokyo'))}"
                                }]
                            }
                            """

                            sh """
                                curl -H "Content-Type: application/json" \\
                                    -d '${payload}' \\
                                    ${DISCORD_WEBHOOK_URL}
                            """
                        }
                    }
                }
            }

        }
        stage('Checkout') {
            steps {
                git branch: "${BRANCH_NAME}",
                url: "${REPO_URL}"
            }
        }
        stage('Build') {
            steps {
                sh './gradlew build'
            }
        }
        stage('Test') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh './gradlew test'
                }
            }
        }
        stage('Configure Git') {
            steps {
                withCredentials([
                    string(credentialsId: 'GitUserName', variable: 'GIT_USER_NAME'),
                    string(credentialsId: 'GitUserEmail', variable: 'GIT_USER_EMAIL')
                ]) {
                    sh '''
                    git config --global user.name "${GIT_USER_NAME}"
                    git config --global user.email "${GIT_USER_EMAIL}"
                    '''
                }
            }
        }
        stage('Tag and Push') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'GitHub-Token', variable: 'GIT_TOKEN')]) {
                        sh '''
                        if [ -d "${REPO_NAME}" ]; then
                            rm -rf ${REPO_NAME}
                        fi
                        if git rev-parse "refs/tags/${TAG_NAME}" >/dev/null 2>&1; then
                            echo "Tag ${TAG_NAME} already exists. Skipping tag creation."
                        else
                            git tag -a "${TAG_NAME}" -m "Jenkins Build #${BUILD_NUMBER}"
                            git push "https://oauth2:${GIT_TOKEN}@github.com/${USERNAME}/${REPO_NAME}.git" --tags
                        fi
                        '''
                    }
                }
            }
        }
        stage('Create Release on GitHub') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'GitHub-Token', variable: 'GIT_TOKEN')]) {
                        def response = httpRequest(
                            acceptType: 'APPLICATION_JSON',
                            contentType: 'APPLICATION_JSON',
                            httpMode: 'POST',
                            url: "https://api.github.com/repos/${USERNAME}/${REPO_NAME}/releases",
                            requestBody: """{
                                "tag_name": "${TAG_NAME}",
                                "name": "Release ${BUILD_NUMBER}",
                                "body": "Release description",
                                "draft": false,
                                "prerelease": false
                            }""",
                            customHeaders: [[name: 'Authorization', value: "token ${GIT_TOKEN}"]]
                        )
                        
                        echo "GitHub Release response content: ${response.content}"
                        
                        def releaseId = new groovy.json.JsonSlurper().parseText(response.content).id
                        echo "GitHub Release ID: ${releaseId}"

                        def jarFilePath = "${JAR_PATH}"
                        if (!fileExists(jarFilePath)) {
                            error "JAR file not found: ${jarFilePath}"
                        }
                        def jarFileBytes = readFile(file: jarFilePath, encoding: 'ISO-8859-1').getBytes('ISO-8859-1')
                        def uploadUrl = "https://uploads.github.com/repos/${USERNAME}/${REPO_NAME}/releases/${releaseId}/assets?name=${jarFilePath.split('/').last()}"

                        def uploadResponse = httpRequest(
                            acceptType: 'APPLICATION_JSON',
                            contentType: 'APPLICATION_OCTETSTREAM',
                            httpMode: 'POST',
                            url: uploadUrl,
                            requestBody: new String(jarFileBytes, 'ISO-8859-1'),
                            customHeaders: [[name: 'Authorization', value: "token ${GIT_TOKEN}"],
                                            [name: 'Content-Type', value: 'application/java-archive']]
                        )
                        echo "GitHub Upload Response: ${uploadResponse}"
                    }
                }
            }
        }
        stage('Discord Result Notification') {
            steps {
                script {
                    if (BUILD_RESULT_NOTIFICATION) {
                        withCredentials([string(credentialsId: 'Discord-Webhook-URL', variable: 'DISCORD_WEBHOOK_URL')]) {
                            def releaseUrl = "https://github.com/${USERNAME}/${REPO_NAME}/releases/tag/${TAG_NAME}"
                            def payload = """
                            {
                                "username": "${WEBHOOK_NAME}",
                                "avatar_url": "${WEBHOOK_AVATAR_URL}",
                                "embeds": [{
                                    "title": "新しいリリースが作成されました",
                                    "description": "**リリース情報**",
                                    "color": ${WEBHOOK_BUILD_RESULT_COLOR},
                                    "author": {
                                        "name": "${USERNAME}",
                                        "icon_url": "https://github.com/${USERNAME}.png",
                                        "url": "https://github.com/${USERNAME}"
                                    },
                                    "thumbnail": {
                                        "url": "${BUILD_RESULT_THUMBNAIL_URL}"
                                    },
                                    "fields": [
                                        {
                                            "name": "リリースタグ",
                                            "value": "${TAG_NAME}",
                                            "inline": true
                                        },
                                        {
                                            "name": "ビルド番号",
                                            "value": "#${BUILD_NUMBER}",
                                            "inline": true
                                        },
                                        {
                                            "name": "リリースURL",
                                            "value": "${releaseUrl}"
                                        }
                                    ],
                                    "timestamp": "${new Date().format("yyyy-MM-dd'T'HH:mm:ss.SSS'Z'", TimeZone.getTimeZone('Asia/Tokyo'))}"
                                }]
                            }
                            """

                            sh """
                                curl -H "Content-Type: application/json" \\
                                    -d '${payload}' \\
                                    ${DISCORD_WEBHOOK_URL}
                            """
                        }
                    }
                }
            }
        }
    }
    post {
        failure {
            script {
                // Check if the failure is not from 'Check Build Interval' stage
                def abortedByIntervalCheck = currentBuild.rawBuild.getCauses().any { cause ->
                    cause.toString().contains('Build aborted: Only')
                }
                
                if (!abortedByIntervalCheck) {
                    withCredentials([string(credentialsId: 'Discord-Webhook-URL', variable: 'DISCORD_WEBHOOK_URL')]) {
                        def payload = """
                        {
                            "username": "${WEBHOOK_NAME}",
                            "avatar_url": "${WEBHOOK_AVATAR_URL}",
                            "embeds": [{
                                "title": "ビルド失敗通知",
                                "description": "ビルドが失敗しました。詳細は [コンソールログ](${BUILD_URL}pipeline-console/ ) を確認してください。",
                                "color": ${WEBHOOK_BUILD_RESULT_FAILDED_COLOR},
                                "thumbnail": {
                                    "url": "${BUILD_FAILED_THUMBNAIL_URL}"
                                },
                                "fields": [
                                    {
                                        "name": "ビルドURL",
                                        "value": "${BUILD_URL}",
                                        "inline": true
                                    },
                                    {
                                        "name": "リポジトリ",
                                        "value": "${REPO_URL}",
                                        "inline": true
                                    }
                                ],
                                "timestamp": "${new Date().format("yyyy-MM-dd'T'HH:mm:ss.SSS'Z'", TimeZone.getTimeZone('Asia/Tokyo'))}"
                            }]
                        }
                        """
                        
                        sh """
                            curl -H "Content-Type: application/json" \\
                                 -d '${payload}' \\
                                 ${DISCORD_WEBHOOK_URL}
                        """
                    }
                }
            }
        }
    }
}