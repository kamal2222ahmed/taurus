@Library('shared-pipeline-library')_

podTemplate(label: 'buildAgent', serviceAccount: 'vmss-jenkins', containers: [
    containerTemplate(name: 'docker', image: 'docker:stable-dind', ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'kubectl', image: 'lachlanevenson/k8s-kubectl:v1.10.3', command: 'cat', ttyEnabled: true)
    ],
    volumes: [
      hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
    ])

node()
{
    try
    {
        stage('Checkout')
        {
            cleanWs()
            scmVars = checkout scm
            commitHash = scmVars.GIT_COMMIT
            isTag =  scmVars.GIT_BRANCH.startsWith("refs/tags/")
            IMAGE_TAG = env.JOB_NAME + "." + env.BUILD_NUMBER
            IMAGE_TAG = IMAGE_TAG.toLowerCase()
        }

        stage("Docker Image Build")
        {
            sh """
                docker build -t ${JOB_NAME} .
                """
        }

        stage("Integration Tests")
        {
            sh """
                docker run -v `pwd`:/bzt-configs -v `pwd`/integr-artifacts:/tmp/artifacts ${JOB_NAME} -sequential examples/all-executors.yml
                """
        }

        stage("Create Artifacts")
        {
            sh """
                sed -ri "s/OS: /Rev: ${commitHash}; OS: /" bzt/cli.py
            """

            if (!isTag) {
                sh """
                sed -ri "s/VERSION = .([^\\"]+)./VERSION = '\\1.${BUILD_NUMBER}'/" bzt/__init__.py
                """
            }

            sh """
                docker run --entrypoint /bzt-configs/build-artifacts.bash -v `pwd`:/bzt-configs ${JOB_NAME} ${BUILD_NUMBER}
                """

            archiveArtifacts artifacts: 'dist/*.whl', fingerprint: true
        }

        stage("Deploy site")
        {
            sh """
                docker build -t deploy-image -f site/Dockerfile.deploy .
                """
            sh """
                mkdir -p scripts
                cp site/deploy-site.sh scripts/deploy-site.sh
                cp site/build-base-site.sh scripts/build-base-site.sh
                cp site/build-snapshot-site.sh scripts/build-snapshot-site.sh
                cp site/Dockerfile* scripts/
                """
            PROJECT_ID="blazemeter-taurus-website-prod"
            withCredentials([file(credentialsId: "${PROJECT_ID}", variable: 'CRED_JSON')]) {
                def WORKSPACE_JSON = 'Google_credentials.json'
                def input = readJSON file: CRED_JSON
                writeJSON file: WORKSPACE_JSON, json: input
                sh """
                    docker run --entrypoint /bzt/scripts/deploy-site.sh \
                    -e KEY_FILE=${WORKSPACE_JSON} \
                    -e PROJECT_ID=${PROJECT_ID} \
                    -e BUILD_NUMBER=${BUILD_NUMBER} \
                    -u root \
                    -v /var/run/docker.sock:/var/run/docker.sock \
                    -v `pwd`:/bzt -t deploy-image \
                    ${isTag}
                    """
            }

        }
    } catch (e) {
         currentBuild.result = "FAILED"
         throw e
    } finally {
         smartSlackNotification(channel: "taurus-dev", buildStatus:currentBuild.result ?: 'SUCCESS')
    }
}
