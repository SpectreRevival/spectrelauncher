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
                                    /*sh "mkdir -p deploy/usr/bin"
                                    sh "mkdir -p deploy/usr/share/applications"
                                    sh "mkdir -p deploy/usr/share/icons/hicolor/256x256/apps"
                                    sh "cp out/build/x64-release-linux/SpectreRevivalLauncher deploy/usr/bin"
                                    sh "cp assets/ico256.png deploy/usr/share/icons/hicolor/256x256/apps/spectre-revival-launcher.png"
                                    sh "cp assets/AppImageDeploy.desktop deploy/usr/share/applications/spectre-launcher.desktop"
                                    sh "linuxdeployqt deploy/usr/share/applications/spectre-launcher.desktop -appimage -unsupported-allow-new-glibc -qmake=out/build/x64-release-linux/vcpkg_installed/x64-linux/tools/Qt6/bin/qmake"
                                    sh "zip -j launcher-linux-x64.zip *.AppImage"*/
                                    echo "not bothering with linuxdeployqt right now since Spectre is windows-only"
                                } else {
                                    bat """
                                        out\\build\\x64-release-win\\vcpkg_installed\\x64-windows\\tools\\qt6\\bin\\windeployqt.exe ^
                                        --release --force ^
                                        out\\build\\x64-release-win\\SpectreRevivalLauncher.exe
                                    """
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
                                    cleanCPPBuildDir("out/build/x64-${BUILD_TYPE}-win", "package-${BUILD_TYPE}-win", BUILD_TYPE == "debug")
				    bat "rmdir /s /q package-release-win\\SpectreRevivalLauncher_autogen"
				    bat "rmdir /s /q package-release-win\\.qt"
                                    archiveArtifacts artifacts: "package-${BUILD_TYPE}-win/**", fingerprint: true
                                } else {
                                    //archiveArtifacts artifacts: "launcher-linux-x64.zip", fingerprint: true
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
