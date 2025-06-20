node {
    env.SONAR_HOST = 'http://52.23.219.98:9000'
    env.SONAR_TOKEN_CREDENTIAL_ID = 'sonar'
    env.NEXUS_URL = 'http://52.23.219.98:8081/repository/maven-snapshots/'
    env.NEXUS_USERNAME = 'admin'
    env.NEXUS_PASSWORD = 'Mubsad321.'
    env.SLACK_WEBHOOK_URL = 'https://hooks.slack.com/services/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'
    env.TOMCAT_URL = 'http://52.23.219.98:8083/manager/text'
    env.TOMCAT_USERNAME = 'admin'
    env.TOMCAT_PASSWORD = 'admin123'
    env.APP_CONTEXT = 'simplecustomerapp-sp' // FIXED: Changed to all lowercase for consistency and SonarQube
    env.GIT_REPO = 'https://github.com/mubeen-hub78/mub_simplecutomerapp.git'
    env.GIT_BRANCH = 'master'

    def buildStatus = 'SUCCESS'
    def slackMessage = ''

    try {
        stage('Git Clone') {
            echo 'Cloning repository...'
            cleanWs()
            git branch: "${env.GIT_BRANCH}", url: "${env.GIT_REPO}"
        }

        stage('Maven Compilation') {
            echo 'Building with Maven inside Docker...'
            def containerProjectRoot = "/app"
            def mavenContainerName = "maven-build-${UUID.randomUUID()}"

            sh """
                docker run --rm \\
                  -v "${env.WORKSPACE}:${containerProjectRoot}" \\
                  -v /var/lib/jenkins/.m2:/root/.m2 \\
                  -w "${containerProjectRoot}" \\
                  --user \$(id -u):\$(id -g) \\
                  maven:3.8.6-eclipse-temurin-17 \\
                  mvn clean compile package -DskipTests
            """
            sh """
                echo "Verifying contents of target/classes after Maven build:"
                if [ -d "${env.WORKSPACE}/target/classes" ]; then
                    echo "target/classes directory found. Listing all files recursively:"
                    find "${env.WORKSPACE}/target/classes" -print -exec ls -ld {} +
                    
                    if find "${env.WORKSPACE}/target/classes" -name "*.class" | grep -q .; then
                        echo "SUCCESS: target/classes contains compiled .class files."
                    else
                        echo "ERROR: target/classes directory exists but NO .class files were found! Maven compilation might have failed to produce class files or the source is empty."
                        exit 1
                    fi
                else
                    echo "ERROR: target/classes directory NOT FOUND after Maven build! This indicates a severe Maven compilation issue or incorrect project structure."
                    exit 1
                fi
                echo "Listing all files in target/ directory for WAR verification:"
                ls -l "${env.WORKSPACE}/target/"
            """
        }

        stage('SonarQube Analysis') {
            echo 'Running SonarQube analysis...'
            withCredentials([string(credentialsId: "${env.SONAR_TOKEN_CREDENTIAL_ID}", variable: 'SONAR_TOKEN')]) {
                def containerProjectRoot = "/app"
                sh """
                    docker run --rm \\
                      -e SONAR_HOST_URL=${env.SONAR_HOST} \\
                      -e SONAR_TOKEN=${SONAR_TOKEN} \\
                      -v "${env.WORKSPACE}:${containerProjectRoot}" \\
                      -w "${containerProjectRoot}" \\
                      --user \$(id -u):\$(id -g) \\
                      sonarsource/sonar-scanner-cli \\
                      -Dsonar.projectKey=${env.APP_CONTEXT} \\
                      -Dsonar.sources=src \\
                      -Dsonar.java.binaries=target/classes \\
                      -Dsonar.working.directory=/app/.scannerwork \\
                      -Dsonar.userHome=/app/.sonar \\
                      -Dsonar.host.url=${env.SONAR_HOST} \\
                      # Removed -Dsonar.login as it's redundant with SONAR_TOKEN env variable
                """
            }
        }

        stage('Nexus Artifactory') {
            echo 'Uploading artifact to Nexus...'
            // The glob pattern now matches the exact casing of the artifactId from pom.xml
            def warFile = findFiles(glob: "target/${env.APP_CONTEXT}-*.war")
            if (warFile.length == 0) {
                error "WAR file not found with expected name 'target/${env.APP_CONTEXT}-*.war'. Please check your pom.xml artifactId or Maven build output."
            }
            def actualWarFile = warFile[0].path

            sh """
                curl -v -u ${env.NEXUS_USERNAME}:${env.NEXUS_PASSWORD} --upload-file ${actualWarFile} \\
                  ${env.NEXUS_URL}${env.APP_CONTEXT}/0.1-SNAPSHOT/${env.APP_CONTEXT}-0.1-SNAPSHOT.war
            """
        }

        stage('Deploy On Tomcat') {
            echo 'Deploying WAR to Tomcat...'
            // The glob pattern now matches the exact casing of the artifactId from pom.xml
            def warFile = findFiles(glob: "target/${env.APP_CONTEXT}-*.war")
            if (warFile.length == 0) {
                error "WAR file not found for deployment with expected name 'target/${env.APP_CONTEXT}-*.war'. Check previous stages."
            }
            def actualWarFile = warFile[0].path

            sh """
                curl -T ${actualWarFile} \\
                  "${env.TOMCAT_URL}/deploy?path=/${env.APP_CONTEXT}&update=true" \\
                  --user ${env.TOMCAT_USERNAME}:${env.TOMCAT_PASSWORD}
            """
            echo "Deployment to Tomcat completed. Access your app at: http://52.23.219.98:8083/${env.APP_CONTEXT}/"
        }

        slackMessage = "✅ *Build SUCCESSFUL* for *${env.APP_CONTEXT}* on *${env.GIT_BRANCH}* branch! 🚀"

    } catch (Exception e) {
        buildStatus = 'FAILURE'
        slackMessage = "❌ *Build FAILED* for *${env.APP_CONTEXT}* on *${env.GIT_BRANCH}* branch! 💥 Error: ${e.message.replaceAll('\\s+', ' ')}"
        echo "Pipeline failed: ${e.message}"
        throw e
    } finally {
        stage('Slack Notification') {
            echo 'Sending Slack notification...'
            def messagePayload = """
                {
                    "text": "${slackMessage}"
                }
            """
            sh """
                set +e
                curl -X POST -H 'Content-type: application/json' \\
                  --data '${messagePayload}' ${env.SLACK_WEBHOOK_URL}
                set -e
            """
        }
        currentBuild.result = buildStatus
    }
}
