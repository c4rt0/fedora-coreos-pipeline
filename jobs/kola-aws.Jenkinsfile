node {
    checkout scm
    // these are script global vars
    pipeutils = load("utils.groovy")
    pipecfg = pipeutils.load_pipecfg()
}

properties([
    pipelineTriggers([]),
    parameters([
      choice(name: 'STREAM',
             choices: pipeutils.get_streams_choices(pipecfg),
             description: 'CoreOS stream to test'),
      string(name: 'VERSION',
             description: 'CoreOS Build ID to test',
             defaultValue: '',
             trim: true),
      string(name: 'ARCH',
             description: 'Target architecture',
             defaultValue: 'x86_64',
             trim: true),
      string(name: 'KOLA_TESTS',
             description: 'Override tests to run',
             defaultValue: "",
             trim: true),
      string(name: 'COREOS_ASSEMBLER_IMAGE',
             description: 'Override the coreos-assembler image to use',
             defaultValue: "quay.io/coreos-assembler/coreos-assembler:main",
             trim: true),
      string(name: 'SRC_CONFIG_COMMIT',
             description: 'The exact config repo git commit to run tests against',
             defaultValue: '',
             trim: true),
    ]),
    buildDiscarder(logRotator(
        numToKeepStr: '100',
        artifactNumToKeepStr: '30'
    )),
    durabilityHint('PERFORMANCE_OPTIMIZED')
])

currentBuild.description = "[${params.STREAM}][${params.ARCH}] - ${params.VERSION}"

def s3_stream_dir = pipeutils.get_s3_streams_dir(pipecfg, params.STREAM)

def stream_info = pipecfg.streams[params.STREAM]

cosaPod(memory: "512Mi", kvm: false,
        image: params.COREOS_ASSEMBLER_IMAGE,
        serviceAccount: "jenkins") {
    timeout(time: 90, unit: 'MINUTES') {
    try {

        lock(resource: "kola-cloud-buildfetch") {
            stage('Fetch Metadata') {
                def commitopt = ''
                if (params.SRC_CONFIG_COMMIT != '') {
                    commitopt = "--commit=${params.SRC_CONFIG_COMMIT}"
                }
                withCredentials([file(variable: 'AWS_CONFIG_FILE',
                                    credentialsId: 'aws-build-upload-config')]) {
                    def (url, ref) = pipeutils.get_source_config_for_stream(pipecfg, params.STREAM)
                    def variant = stream_info.variant ? "--variant ${stream_info.variant}" : ""
                    shwrap("""
                    cosa init --branch ${ref} ${commitopt} ${variant} ${url}
                    time -v cosa buildfetch --artifact=ostree --build=${params.VERSION} \
                        --arch=${params.ARCH} --url=s3://${s3_stream_dir}/builds
                    """)
                }
            }
        }

        withCredentials([file(variable: 'AWS_CONFIG_FILE',
                              credentialsId: 'aws-kola-tests-config')]) {
            // A few independent tasks that can be run in parallel
            def parallelruns = [:]
            try {
                parallelruns['Kola:Full'] = {
                    kola(cosaDir: env.WORKSPACE, parallel: 5,
                        build: params.VERSION, arch: params.ARCH,
                        extraArgs: params.KOLA_TESTS,
                        skipBasicScenarios: true,
                        platformArgs: '-p=aws --aws-region=us-east-1',
                        skipKolaTags: stream_info.skip_kola_tags)
                }

                if (params.ARCH == "x86_64") {
                    def tests = params.KOLA_TESTS
                    if (tests == "") {
                        tests = "basic"
                    }
                    parallelruns['Kola:Xen'] = {
                        // https://github.com/coreos/fedora-coreos-tracker/issues/997
                        // Run this test on i3.large so we can also run ext.config.platforms.aws.nvme
                        // to verify access to instance storage nvme disks works
                        // https://github.com/coreos/fedora-coreos-tracker/issues/1306
                        // Also add in the ext.config.platforms.aws.assert-xen test just
                        // to sanity check we are on a Xen instance.
                        def xen_tests = tests
                        if (xen_tests == "basic") {
                            xen_tests = "basic ext.config.platforms.aws.nvme ext.config.platforms.aws.assert-xen"
                        }
                        kola(cosaDir: env.WORKSPACE,
                            build: params.VERSION, arch: params.ARCH,
                            extraArgs: "--tag aws-xen-test " + xen_tests,
                            skipUpgrade: true,
                            marker: "xen",
                            platformArgs: '-p=aws --aws-region=us-east-1 --aws-type=i3.large',
                            skipKolaTags: stream_info.skip_kola_tags)
                    }
                    parallelruns['Kola:Intel-Ice-Lake'] = {
                        // https://github.com/coreos/fedora-coreos-tracker/issues/1004
                        kola(cosaDir: env.WORKSPACE,
                            build: params.VERSION, arch: params.ARCH,
                            extraArgs: tests,
                            skipUpgrade: true,
                            marker: "intel-ice-lake",
                            platformArgs: '-p=aws --aws-region=us-east-1 --aws-type=m6i.large',
                            skipKolaTags: stream_info.skip_kola_tags)
                    }
                } else if (params.ARCH == "aarch64") {
                    def tests = params.KOLA_TESTS
                    if (tests == "") {
                        tests = "basic"
                    }
                    parallelruns['Kola:Graviton3'] = {
                        // https://aws.amazon.com/ec2/instance-types/c7g/
                        kola(cosaDir: env.WORKSPACE,
                            build: params.VERSION, arch: params.ARCH,
                            extraArgs: tests,
                            skipUpgrade: true,
                            marker: "graviton3",
                            platformArgs: '-p=aws --aws-region=us-east-1 --aws-type=c7g.xlarge',
                            skipKolaTags: stream_info.skip_kola_tags)
                    }
                }

                // process this batch
                parallel parallelruns
            } finally {
                stage('Garbage Collection') {
                    shwrap("""
                    ore aws gc --debug                         \
                        --credentials-file=\${AWS_CONFIG_FILE} \
                        --region=us-east-1
                    """)
                }
            }

        }

        currentBuild.result = 'SUCCESS'

    } catch (e) {
        currentBuild.result = 'FAILURE'
        throw e
    } finally {
        if (currentBuild.result != 'SUCCESS') {
            pipeutils.trySlackSend(message: ":aws: kola-aws #${env.BUILD_NUMBER} <${env.BUILD_URL}|:jenkins:> <${env.RUN_DISPLAY_URL}|:ocean:> [${params.STREAM}][${params.ARCH}] (${params.VERSION})")
        }
    }
}} // cosaPod and timeout finish here
