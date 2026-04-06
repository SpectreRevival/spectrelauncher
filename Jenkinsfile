pipeline {
    agent none

    stages {
        stage('Build Matrix') {
            matrix {
                axes {
                    axis {
                        name 'OS'
                        values 'windows', 'linux'
                    }
                    axis {
                        name 'BUILD_TYPE'
                        values 'debug', 'release'
                    }
                }

                agent { label "${OS}" }
                options {
                    throttle(['RamIntensiveJob'])
                }

                stages {
                    stage('Checkout') {
                        steps {
                            checkout scm
                            script {
                                if (isUnix()) {
                                    sh "git submodule sync --recursive"
                                    sh "git submodule update --init --recursive"
                                } else {
                                    bat "git submodule sync --recursive"
                                    bat "git submodule update --init --recursive"
                                }
                            }
                        }
                    }

                    stage('Get vcpkg commit') {
                        steps {
                            script {
                                def sha
                                if (isUnix()) {
                                    sha = sh(script: "cd vcpkg && git rev-parse HEAD", returnStdout: true).trim()
                                } else {
                                    sha = bat(script: "cd vcpkg && git rev-parse HEAD", returnStdout: true).trim()
                                }
                                env.VCPKG_SHA = sha
                            }
                        }
                    }

                    stage('Configure') {
                        steps {
                            script {
                                if (env.OS == 'windows') {
                                    bat """
                                        call \"C:\\Program Files\\Microsoft Visual Studio\\18\\Community\\VC\\Auxiliary\\Build\\vcvarsall.bat\" x64
                                        cmake --preset x64-${BUILD_TYPE}-win
                                    """
                                } else {
                                    sh "cmake --preset x64-${BUILD_TYPE}-linux"
                                }
                            }
                        }
                    }

                    stage('Build') {
                        steps {
                            script {
                                if (env.OS == 'windows') {
                                    bat """
                                        call \"C:\\Program Files\\Microsoft Visual Studio\\18\\Community\\VC\\Auxiliary\\Build\\vcvarsall.bat\" x64
                                        cmake --build out/build/x64-${BUILD_TYPE}-win
                                    """
                                } else {
                                    sh "cmake --build out/build/x64-${BUILD_TYPE}-linux"
                                }
                            }
                        }
                    }

                    stage("Deploy qt application"){
                        when {
                            expression { BUILD_TYPE == 'release' }
                        }
                        steps {
                            script {
                                if(isUnix()){
                                    sh "mkdir -p deploy/usr/bin"
                                    sh "mkdir -p deploy/usr/share/applications"
                                    sh "mkdir -p deploy/usr/share/icons/hicolor/256x256/apps"
                                    sh "cp build/SpectreRevivalLauncher deploy/usr/bin"
                                    sh "cp assets/ico256.png deploy/usr/share/icons/hicolor/256x256/apps/spectre-revival-launcher.png"
                                    sh "cp assets/AppImageDeploy.desktop deploy/usr/share/applications/spectre-launcher.desktop"
                                    sh "linuxdeployqt deploy/usr/share/applications/spectre-launcher.desktop -appimage"
                                    sh "zip -j launcher-linux-x64.zip *.AppImage"
                                } else {
                                    bat """
                                        out\\build\\x64-release-win\\vcpkg_installed\\x64-windows\\tools\\qt6\\bin\\windeployqt.exe ^
                                        --release --force ^
                                        --dir deployed ^
                                        out\\build\\x64-release-win\\SpectreRevivalLauncher.exe
                                    """
                                    powershell "Compress-Archive -Path 'deployed\\*' -DestinationPath 'launcher-win-x64.zip' -Force"
                                }
                            }
                        }
                    }

                    stage('Archive') {
                        when {
                            expression { BUILD_TYPE == 'release' }
                        }
                        steps {
                            script {
                                if (env.OS == 'windows') {
                                    archiveArtifacts artifacts: "launcher-win-x64.zip", fingerprint: true
                                } else {
                                    archiveArtifacts artifacts: "launcher-linux-x64.zip", fingerprint: true
                                }
                            }
                        }
                    }
                }
            }
        }

        stage("code analysis") {
            parallel {
                stage("Code formatter") {
                    agent { label 'linux' }
                    steps {
                        script {
                            stage("Checkout") {
                                checkout scm
                            }
                            stage("Run formatter") {
                                sh """
                                    FILES=\$(find ./src -type f -regex '.*\\.cpp\$\\|.*\\.h\$')
                                    clang-format -i \$FILES
                                """
                            }
                            stage("Create diff") {
                                sh """
                                    if ! git diff --quiet; then
                                        git diff > clang-format.patch
                                        echo \"Patch created, apply the patch from the artifacts section to fix\"
                                    else
                                        echo \"No changes required\"
                                    fi
                                """
                            }
                            stage("Upload diff if exists") {
                                if (fileExists('clang-format.patch')) {
                                    archiveArtifacts artifacts: 'clang-format.patch', fingerprint: true
                                }
                            }
                            stage("Fail if diff exists") {
                                if (fileExists('clang-format.patch')) {
                                    sh "exit 1"
                                }
                            }
                        }
                    }
                }

                stage("Code linter") {
                    agent { label 'linux' }
                    options {
                        throttle(['RamIntensiveJob'])
                    }
                    steps {
                        script {
                            stage("Checkout") {
                                checkout scm
                                sh "git submodule sync --recursive"
                                sh "git submodule update --init --recursive"
                            }
                            stage("Configure") {
                                sh "cmake --preset x64-debug-linux"
                            }
                            stage("Run linter") {
                                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE'){
                                sh """
                                    FILES=\$(find ./src -type f -regex '.*\\.cpp\$\\|.*\\.h\$')
                                    run-clang-tidy \$FILES -fix -p out/build/x64-debug-linux -extra-arg=-Werror
                                """
                                }
                            }
                            stage("Create diff") {
                                sh """
                                    if ! git diff --quiet; then
                                        git diff > clang-format.patch
                                        echo \"Patch created, apply the patch from the artifacts section to fix\"
                                    else
                                        echo \"No changes required\"
                                    fi
                                """
                            }
                            stage("Upload diff if exists") {
                                if (fileExists('clang-format.patch')) {
                                    archiveArtifacts artifacts: 'clang-format.patch', fingerprint: true
                                }
                            }
                            stage("Fail if diff exists") {
                                if (fileExists('clang-format.patch')) {
                                    sh "exit 1"
                                }
                            }
                        }
                    }
                }
            }
        }
    }
    post {
        always {
            node('linux'){
                step([$class: 'GitHubCommitStatusSetter', contextSource: [$class: 'ManuallyEnteredCommitContextSource', context: 'jenkins/build-status'], reposSource: [$class: 'ManuallyEnteredRepositorySource', url: 'https://github.com/SpectreRevival/spectrelauncher']])
            }
        }
    }
}
