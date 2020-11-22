def pys = [
    [name: 'Python 3.9', docker:'python:3.9-buster', tox:'py39', main: false],
    [name: 'Python 3.8', docker:'python:3.8-buster', tox:'py38,flake8', main: true],
    [name: 'Python 3.7', docker:'python:3.7-buster', tox:'py37', main: false],
    [name: 'Python 3.6', docker:'python:3.6-buster', tox:'py36', main: false],
    [name: 'Python 3.5', docker:'python:3.5-buster', tox:'py35', main: false],
]

properties([
    durabilityHint('PERFORMANCE_OPTIMIZED'),
    buildDiscarder(logRotator(numToKeepStr: '100')),
])

Map tasks = [failFast: true]

pys.each { py ->
    tasks[py.name] = {
        node {
            def image

            stage("Prepare docker $py.name") {
                dir('dockerbuild') {
                    deleteDir()
                    docker.image(py.docker).pull()
                    buildDockerfile(py.docker)
                    image = docker.build("dosage-$py.docker")
                }
            }

            stage("Build $py.name") {
                image.inside {
                    checkout scm
                    sh '''
                        git clean -fdx
                        git fetch --tags
                    '''

                    warnError('tox failed') {
                        sh "tox -e $py.tox"
                    }

                    if (py.main) {
                        sh """
                            python setup.py sdist bdist_wheel
                        """
                    }
                }

                if (py.main) {
                    archiveArtifacts artifacts: 'dist/*', fingerprint: true
                    stash includes: 'dist/*.tar.gz', name: 'bin'
                    dir('.tox') {
                        stash includes: 'allure-*/**', name: 'allure'
                    }
                    def buildVer = findFiles(glob: 'dist/*.tar.gz')[0].name.replaceFirst(/\.tar\.gz$/, '')
                    currentBuild.description = buildVer

                    publishCoverage calculateDiffForChangeRequests: true,
                        sourceFileResolver: sourceFiles('STORE_LAST_BUILD'),
                        adapters: [
                            coberturaAdapter('.tox/cov-*.xml')
                        ]

                    recordIssues sourceCodeEncoding: 'UTF-8',
                        referenceJobName: 'dosage/master',
                        tool: flake8(pattern: '.tox/flake8.log', reportEncoding: 'UTF-8')
                }
                junit '.tox/junit-*.xml'
            }
        }
    }
}

timestamps {
    ansiColor('xterm') {
        parallel(tasks)
        windowsBuild()
        processAllure()
    }
}

def buildDockerfile(image) {
    def uid = sh(returnStdout: true, script: 'id -u').trim()
    writeFile file: 'Dockerfile', text: """
    FROM $image
    RUN pip install tox
    RUN useradd -mu $uid dockerjenkins
    """
}

def windowsBuild() {
    stage('Windows binary') {
        warnError('windows build failed') {
            node {
                windowsBuildCommands()
            }
        }
    }
}

def windowsBuildCommands() {
    deleteDir()
    unstash 'bin'
    def img = docker.image('tobix/pywine')
    img.pull()
    img.inside {
        sh '''
            . /opt/mkuserwineprefix
            tar xvf dist/dosage-*.tar.gz
            cd dosage-*
            xvfb-run sh -c "
                wine py -m pip install -e .[css] &&
                cd scripts &&
                wine py -m PyInstaller -y dosage.spec;
                wineserver -w" 2>&1 | tee log.txt
        '''
        archiveArtifacts '*/scripts/dist/*'
    }
}

def processAllure() {
    stage('Allure report') {
        warnError('allure report failed') {
            deleteDir()
            unstash 'allure'
            docker.image('tobix/allure-cli').inside {
                sh 'allure generate allure-*'
            }
            publishHTML reportDir: 'allure-report', reportFiles: 'index.html', reportName: 'Allure Report'
        }
    }
}

// vim: set ft=groovy:
