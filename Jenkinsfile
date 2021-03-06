def DOCKER_IMG = "registry.fedoraproject.org/fedora:28"
def DOCKER_ARGS = "--net=host -v /srv:/srv --privileged"

// this var conveniently refers to a location on the server as well as the local dir we sync to/from
def repo = "${env.ARTIFACT_SERVER_DIR}/repo"

if (env.BUILD_TYPE != 'origin' && env.BUILD_TYPE != 'rhcos') {
    assert false
}
def manifest = "host-${env.BUILD_TYPE}.json"

node(env.NODE) {
    checkout scm

    docker.image(DOCKER_IMG).inside(DOCKER_ARGS) {
        stage("Provision") {
            sh "dnf install -y git rpm-ostree rsync openssh-clients"
            sh "cp RPM-GPG-* /etc/pki/rpm-gpg/"
        }

        stage("Sync In") {
            withCredentials([sshUserPrivateKey(credentialsId: env['ARTIFACT_SSH_CREDS_ID'],
                                               keyFileVariable: 'KEY_FILE')]) {
                sh """
                    # a few idempotent commands for bootstrapping
                    mkdir -p ${repo}
                    ostree init --repo=${repo} --mode=archive

                    rsync -Hrlpt --stats \
                        -e 'ssh -i ${env.KEY_FILE} \
                                -o UserKnownHostsFile=/dev/null \
                                -o StrictHostKeyChecking=no' \
                        ${env.ARTIFACT_SERVER}:${repo}/ ${repo}
                """
            }
        }

        stage("Check for Changes") {
            sh "rm -f build.stamp"
            sh "rpm-ostree compose tree --dry-run --repo=${repo} --touch-if-changed=build.stamp ${manifest}"
        }

        if (!fileExists('build.stamp')) {
            currentBuild.result = 'SUCCESS'
            return
        }

        stage("Compose Tree") {
            sh "rpm-ostree compose tree --repo=${repo} ${manifest}"
        }

        /*
        stage("Build Cloud Images") {
            sh "make cloud TYPE=qcow2"
        }
        */

        stage("Sync Out") {
            withCredentials([sshUserPrivateKey(credentialsId: env['ARTIFACT_SSH_CREDS_ID'],
                                               keyFileVariable: 'KEY_FILE')]) {
                sh """
                    ostree prune --repo=${repo} --keep-younger-than='30 days ago' --refs-only
                    ostree summary --repo=${repo} --update
                    rsync -Hrlpt --stats --delete --delete-after \
                        -e 'ssh -i ${env.KEY_FILE} \
                                -o UserKnownHostsFile=/dev/null \
                                -o StrictHostKeyChecking=no' \
                        ${repo}/ ${env.ARTIFACT_SERVER}:${repo}
                """
            }
        }
    }
}
