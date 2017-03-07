#!/usr/bin/env groovy

/**
 * Build openwrt project
 *
 * Configure as needed, pull in the feeds and run the build. Allows
 * overridding feeds listed using build parameters.
 * 
 * Note this script needs some extra Jenkins permissions to be
 * approved.
 */
// TODO be careful with using dir() until JENKINS-33510 is fixed

def customFeeds = [
]
def feedParams = []
for (feed in customFeeds) {
    feedParams.add(string(defaultValue: '', description: 'Branch/commmit/PR to override feed. \
        (Can be a PR using PR-< id >)', name: "OVERRIDE_${feed[0].toUpperCase()}"))
}

properties([
    buildDiscarder(logRotator(numToKeepStr: '10')),
    parameters([
        booleanParam(defaultValue: false,
            description: 'Build extra tools such as toolchain, SDK and Image builder',
            name: 'BUILD_TOOLS'),
        booleanParam(defaultValue: false, description: 'Build *all* packages for opkg',
            name: 'ALL_PACKAGES'),
        stringParam(defaultValue: '', description: 'Set version, if blank job number will be used.',
            name: 'VERSION'),
        stringParam(defaultValue: '',
            description: 'Branch to use for Boardfarm',
            name: 'OVERRIDE_BOARDFARM'),
    ] + feedParams)
])

node('docker && imgtec') {  // Only run on internal slaves as build takes a lot of resources
    def docker_image

    stage('Prepare Docker container') {
        // Setup a local docker container to run build on this slave
        // TODO for now have manually setup on slaves
        docker_image = docker.image "imgtec/creator-builder:latest"
    }

    // TODO reference repo has to be manually setup on all build slaves right now
    docker_image.inside("-v ${WORKSPACE}/../../reference-repos:${WORKSPACE}/../../reference-repos:ro") {
        stage('Configure') {
            // Checkout a clean version of the repo using reference repo to save bandwidth/time
            checkout([$class: 'GitSCM',
                userRemoteConfigs: scm.userRemoteConfigs,
                branches: scm.branches,
                doGenerateSubmoduleConfigurations: scm.doGenerateSubmoduleConfigurations,
                submoduleCfg: scm.submoduleCfg,
                browser: scm.browser,
                gitTool: scm.gitTool,
                extensions: scm.extensions + [
                    [$class: 'CleanCheckout'],
                    [$class: 'PruneStaleBranch'],
                    [$class: 'CloneOption', honorRefspec: true, reference: "${WORKSPACE}/../../reference-repos/lede-source.git"],
                ],
            ])

            // Versioning
            sh "echo ${params.VERSION?.trim() ?: 'j' + env.BUILD_NUMBER} > version"

            // Default config
            sh 'echo \'' \
             + 'CONFIG_TARGET_pistachio=y\n' \
             + 'CONFIG_TARGET_MULTI_PROFILE=y\n' \
             + 'CONFIG_TARGET_ALL_PROFILES=y\n' \
             + 'CONFIG_TARGET_PER_DEVICE_ROOTFS=y\n' \
             + 'CONFIG_ALL_NONSHARED=y\n' \
             + '\' > .config'

            // Add development config
            echo 'Enabling development config'
            sh 'echo \'' \
             + 'CONFIG_DEVEL=y\n' \
             + 'CONFIG_LOCALMIRROR=\"https://downloads.creatordev.io/pistachio/marduk/dl\"\n' \
             + '\' >> .config'

            // Build tools/sdks
            if (params.BUILD_TOOLS) {
                echo 'Enabling toolchain, image builder and sdk creation'
                sh 'echo \'' \
                 + 'CONFIG_IB=y\n' \
                 + '# CONFIG_IB_STANDALONE is not set\n' \
                 + 'CONFIG_SDK=y\n' \
                 + '\' >> .config'
            }

            // Build all (for opkg)
            if (params.ALL_PACKAGES){
                echo 'Enabling all user and kernel packages'
                sh 'echo \'' \
                 + 'CONFIG_ALL=y\n' \
                 + '\' >> .config'
            }

            // Boardfarm-able
            // TODO work out which ones we actually have
            echo 'Enabling usb ethernet adapters modules (for boardfarm testing)'
            sh 'echo \'' \
             + 'CONFIG_PACKAGE_kmod-usb-net=y\n' \
             + 'CONFIG_PACKAGE_kmod-usb-net-asix=y\n' \
             + 'CONFIG_PACKAGE_kmod-usb-net-cdc-eem=y\n' \
             + 'CONFIG_PACKAGE_kmod-usb-net-cdc-ether=y\n' \
             + 'CONFIG_PACKAGE_kmod-usb-net-cdc-mbim=y\n' \
             + 'CONFIG_PACKAGE_kmod-usb-net-cdc-ncm=y\n' \
             + 'CONFIG_PACKAGE_kmod-usb-net-cdc-subset=y\n' \
             + 'CONFIG_PACKAGE_kmod-usb-net-dm9601-ether=y\n' \
             + 'CONFIG_PACKAGE_kmod-usb-net-hso=y\n' \
             + 'CONFIG_PACKAGE_kmod-usb-net-huawei-cdc-ncm=y\n' \
             + 'CONFIG_PACKAGE_kmod-usb-net-ipheth=y\n' \
             + 'CONFIG_PACKAGE_kmod-usb-net-kalmia=y\n' \
             + 'CONFIG_PACKAGE_kmod-usb-net-kaweth=y\n' \
             + 'CONFIG_PACKAGE_kmod-usb-net-mcs7830=y\n' \
             + 'CONFIG_PACKAGE_kmod-usb-net-pegasus=y\n' \
             + 'CONFIG_PACKAGE_kmod-usb-net-qmi-wwan=y\n' \
             + 'CONFIG_PACKAGE_kmod-usb-net-rndis=y\n' \
             + 'CONFIG_PACKAGE_kmod-usb-net-rtl8150=y\n' \
             + 'CONFIG_PACKAGE_kmod-usb-net-rtl8152=y\n' \
             + 'CONFIG_PACKAGE_kmod-usb-net-sierrawireless=y\n' \
             + 'CONFIG_PACKAGE_kmod-usb-net-smsc95xx=y\n' \
             + '\' >> .config'

            // Add all required feeds to default config
            for (feed in customFeeds) {
                sh "grep -q 'src-.* ${feed[0]} .*' feeds.conf.default || \
                    echo 'src-git ${feed[0]} ${feed[2]}/${feed[1]}.git' >> feeds.conf.default"
            }

            // If specified override each feed with local clone
            sh 'cp feeds.conf.default feeds.conf'
            for (feed in customFeeds) {
                if (params."OVERRIDE_${feed[0].toUpperCase()}"?.trim()){
                    dir("feed-${feed[1]}") {
                        checkout([
                            $class: 'GitSCM',
                            branches: [[name: env."OVERRIDE_${feed[0].toUpperCase()}"]],
                            userRemoteConfigs: [[
                                refspec: '+refs/pull/*/head:refs/remotes/origin/PR-* \
                                    +refs/heads/*:refs/remotes/origin/*',
                                url: "${feed[2]}/${feed[1]}.git"
                            ]]
                        ])
                    }
                    sh "sed -i 's|^src-git ${feed[0]} .*|src-link ${feed[0]} ../feed-${feed[1]}|g' feeds.conf"
                }
            }
            sh 'cat feeds.conf.default feeds.conf .config'
            sh 'scripts/feeds update -a && scripts/feeds install -a'
            sh 'rm feeds.conf'
            sh 'make defconfig'
        }

        stage('Build') {
            // Add opkg signing key
            withCredentials([
                [$class: 'FileBinding', credentialsId: 'opkg-build-private-key', variable: 'PRIVATE_KEY'],
                [$class: 'FileBinding', credentialsId: 'opkg-build-public-key', variable: 'PUBLIC_KEY'],
            ]){
                try {
                    // TODO: LEDE build system requires specific key name and doesn't follow symlinks
                    sh "cp ${env.PRIVATE_KEY} ${WORKSPACE}/key-build"
                    sh "cp ${env.PUBLIC_KEY} ${WORKSPACE}/key-build.pub"

                    sh "make -j8 IGNORE_ERRORS=m || make -j1 IGNORE_ERRORS=m V=s"
                }
                finally {
                    sh "rm ${WORKSPACE}/key-build*"
                }
            }
        }

        stage('Upload') {
            archiveArtifacts 'bin/targets/**'
            archiveArtifacts 'bin/packages/**'
            deleteDir()  // clean up the workspace to save space
        }

        stage('Integration test') {
            //TODO lede has no releases for our arch yet so always force override
            build job: "../boardfarm/master", parameters: [
                booleanParam(name: 'OVERRIDE_PACKAGES', value: true),
                string(name: 'IMAGE_PATH', value: BUILD_URL + \
                    "artifact/bin/targets/pistachio/generic/lede-pistachio-marduk-squashfs-factory.ubi"),
                string(name: 'PACKAGES_PATH', value: BUILD_URL + "artifact/bin"),
            ]
        }
    }
}
