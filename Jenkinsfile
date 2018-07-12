properties([
        buildDiscarder(logRotator(numToKeepStr: '5', artifactNumToKeepStr: '5')),
        disableConcurrentBuilds()
])

echo env.BRANCH_NAME

if (env.BRANCH_NAME == 'test') {
    echo 'test branch'
    currentBuild.result = 'SUCCESS'
    return
}
else {
    
    podTemplate(label: 'jenkins-slave', name: 'jenkins-slave', cloud: 'hcm-cicd') {
        node('jenkins-slave') {

            def PROJECT_ARTIFACT_ID = ''
            def PROJECT_VERSION = ''
            String PROJECT_CONTAINER_TAG = ''
            String PROJECT_CONTAINER_NAME = ''
            String RELEASE_BRANCH = "v\\d*\\.\\d*\\.\\d*"
            try {
                stage('prepare') {
                    echo "Building branch: ${env.BRANCH_NAME}"
                    checkout scm

                    PROJECT_ARTIFACT_ID = sh(
                            script: 'mvn -q -Dexec.executable="echo" -Dexec.args=\'${project.artifactId}\' --non-recursive exec:exec',
                            returnStdout: true
                    ).trim()

                    PROJECT_VERSION = sh(
                            script: 'mvn -q -Dexec.executable="echo" -Dexec.args=\'${project.version}\' --non-recursive exec:exec | tr -d "\\-SNAPSHOT"',
                            returnStdout: true
                    ).trim()

                    PROJECT_CONTAINER_NAME = "cicd-dashboard"

                    if (env.BRANCH_NAME.toString().matches(RELEASE_BRANCH)) {
                        if (env.BRANCH_NAME.toString().substring(1) == PROJECT_VERSION) {
                            BUILD_NUMBER = "Final"
                            PROJECT_CONTAINER_TAG = PROJECT_VERSION
                        } else {
                            error("Branch name [${env.BRANCH_NAME}] does not match Maven project version [${PROJECT_VERSION}]. Branch name and Maven version must agree e.g. branch name \'v1.1.0\' matches Maven version 1.1.0")
                        }
                    } else {
                        PROJECT_CONTAINER_TAG = "${PROJECT_VERSION}-${BUILD_NUMBER}"
                    }


                    echo "artifactId = ${PROJECT_ARTIFACT_ID}"
                    echo "version = ${PROJECT_VERSION}"
                    echo "buildNumber = ${BUILD_NUMBER}"
                    echo "containerTag = ${PROJECT_CONTAINER_TAG}"
                    echo "containerName = ${PROJECT_CONTAINER_NAME}"
                }

                stage('test') {
                    sh "mvn clean test"
                }

                stage('build jenkins plugin') {
                    dir('plugin') {
                        sh "mvn clean package"
                        archiveArtifacts artifacts: '**/target/*.hpi', onlyIfSuccessful: true
                    }
                }

                stage('build service') {
                    sh """mvn -B -U install -Pproduction \
                        -Dmaven.wagon.http.ssl.insecure=true \
                        -DskipTests \
                        -Dcontainer.name=${PROJECT_CONTAINER_NAME} \
                        -Dcontainer.version=${PROJECT_CONTAINER_TAG} \
                        -Dbuild.number=${BUILD_NUMBER} \
                    """
                }

                // DEVELOP branch builds promote to T1 repository
                if (env.BRANCH_NAME == 'develop') {
                    stage('publish') {
                        withCredentials([usernamePassword(credentialsId: 'cicd-docker', passwordVariable: 'T1_DOCKER_REGISTRY_PASSWORD', usernameVariable: 'T1_DOCKER_REGISTRY_USERNAME')]) {
                            sh "docker login \${T1_DOCKER_REGISTRY_HOST}:\${T1_DOCKER_REGISTRY_PORT} -u=\${T1_DOCKER_REGISTRY_USERNAME} -p=\${T1_DOCKER_REGISTRY_PASSWORD}"
                            sh "docker tag ${PROJECT_CONTAINER_NAME}:latest \${T1_DOCKER_REGISTRY_HOST}:\${T1_DOCKER_REGISTRY_PORT}/cicd/${PROJECT_CONTAINER_NAME}:${PROJECT_CONTAINER_TAG}"
                            sh "docker tag ${PROJECT_CONTAINER_NAME}:latest \${T1_DOCKER_REGISTRY_HOST}:\${T1_DOCKER_REGISTRY_PORT}/cicd/${PROJECT_CONTAINER_NAME}:latest"
                            sh "docker push \${T1_DOCKER_REGISTRY_HOST}:\${T1_DOCKER_REGISTRY_PORT}/cicd/${PROJECT_CONTAINER_NAME}:${PROJECT_CONTAINER_TAG}"
                            sh "docker push \${T1_DOCKER_REGISTRY_HOST}:\${T1_DOCKER_REGISTRY_PORT}/cicd/${PROJECT_CONTAINER_NAME}:latest"
                        }
                    }
                }

                if (env.BRANCH_NAME == 'develop') {
                    stage('archive') {
                        GIT_COMMIT = sh(
                                script: 'git rev-parse HEAD',
                                returnStdout: true
                        ).trim()

                        GIT_URL = sh(
                                script: 'git config --get remote.origin.url',
                                returnStdout: true
                        ).trim()

                        dir('target') {
                            sh "echo \"version=${PROJECT_VERSION}\" > ${PROJECT_ARTIFACT_ID}.manifest"
                            sh "echo \"build=${BUILD_NUMBER}\" >> ${PROJECT_ARTIFACT_ID}.manifest"
                            sh "echo \"container.org=${T1_DOCKER_REGISTRY_ORG}\" >> ${PROJECT_ARTIFACT_ID}.manifest"
                            sh "echo \"container.name=${PROJECT_CONTAINER_NAME}\" >> ${PROJECT_ARTIFACT_ID}.manifest"
                            sh "echo \"container.tag=${PROJECT_CONTAINER_TAG}\" >> ${PROJECT_ARTIFACT_ID}.manifest"
                            sh "echo \"container.image=${T1_DOCKER_REGISTRY_HOST}:${T1_DOCKER_REGISTRY_PORT}/cicd/${PROJECT_CONTAINER_NAME}:${PROJECT_CONTAINER_TAG}\" >> ${PROJECT_ARTIFACT_ID}.manifest"
                            sh "echo \"jenkins.build=${BUILD_NUMBER}\" >> ${PROJECT_ARTIFACT_ID}.manifest"
                            sh "echo \"jenkins.build.url=${BUILD_URL}\" >> ${PROJECT_ARTIFACT_ID}.manifest"
                            sh "echo \"git.commit=${GIT_COMMIT}\" >> ${PROJECT_ARTIFACT_ID}.manifest"
                            sh "echo \"git.url=${GIT_URL}\" >> ${PROJECT_ARTIFACT_ID}.manifest"
                        }

                        archiveArtifacts artifacts: '**/target/*.manifest', onlyIfSuccessful: true
                    }
                }

                if (env.BRANCH_NAME == 'develop') {
                    stage('deploy') {
                        withCredentials([sshUserPrivateKey(credentialsId: 'hcm-cicd-root-private-key', keyFileVariable: 'SSH_KEY', passphraseVariable: 'SSH_PASSPHRASE', usernameVariable: 'SSH_USER')]) {
                            echo "Deploying Container"
                            sh "ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -i ${SSH_KEY} root@hcm-cicd-master-1.hpeswlab.net \"kubectl delete pod \\\$(kubectl get pods -n cicd | grep cicd-dashboard | awk {'print \\\$1'}) -n cicd\""
                        }
                    }
                }
                currentBuild.result = 'SUCCESS'
                deleteDir()
            } catch (e) {
                currentBuild.result = 'FAILURE'
                throw e
            }
        }
    }
}
