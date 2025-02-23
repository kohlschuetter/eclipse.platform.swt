/*******************************************************************************
 * Copyright (c) 2021, 2024 Red Hat Inc. and others.
 *
 * This program and the accompanying materials
 * are made available under the terms of the Eclipse Public License 2.0
 * which accompanies this distribution, and is available at
 * https://www.eclipse.org/legal/epl-2.0/
 *
 * SPDX-License-Identifier: EPL-2.0
 *
 * Contributors:
 *     Mickael Istria (Red Hat Inc.) - initial API and implementation
 *     Hannes Wellmann - Build SWT-natives as part of master- and verification-builds
 *     Hannes Wellmann - Move SWT native binaries in this repository using Git-LFS
 *     Hannes Wellmann - Streamline entire SWT build and replace ANT-scripts by Maven, Jenkins-Pipeline and single-source Java scripts
  *******************************************************************************/

def nativeBuildAgent(String platform, Closure body) {
	def final nativeBuildStageName = 'Build SWT-native binaries'
	if (platform == 'gtk.linux.x86_64') {
		return podTemplate(yaml: '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: "swtbuild"
    image: "eclipse/platformreleng-centos-swt-build:8"
    imagePullPolicy: "Always"
    resources:
      limits:
        memory: "4096Mi"
        cpu: "2000m"
      requests:
        memory: "512Mi"
        cpu: "1000m"
    command:
    - cat
    tty: true
    volumeMounts:
    - name: tools
      mountPath: /opt/tools
  volumes:
  - name: tools
    persistentVolumeClaim:
      claimName: tools-claim-jiro-releng
''') { node(POD_LABEL) { stage(nativeBuildStageName) { container('swtbuild') { body() } } } }
	} else {
		return node('swt.natives-' + platform) { stage(nativeBuildStageName) { body() } }
	}
}

/** Returns the download URL of the JDK against whoose C headers (in the 'include/' folder) and native libaries the SWT natives are compiled.*/
def getNativeJdkUrl(String os, String arch){ // To update the used JDK version update the URL template below
	return "https://download.eclipse.org/justj/jres/17/downloads/20230428_1804/org.eclipse.justj.openjdk.hotspot.jre.minimal.stripped-17.0.7-${os}-${arch}.tar.gz"
}

def getLatestGitTag() {
	return sh(script: 'git describe --abbrev=0 --tags --match v[0-9][0-9][0-9][0-9]*', returnStdout: true).strip()
}

def getSWTVersions() { // must be called from the repository root
	def props = readProperties(file: 'bundles/org.eclipse.swt/Eclipse SWT/common/library/make_common.mak')
	props['new_rev'] = props['rev'].toInteger() + 1
	props['swt_version'] = props['maj_ver'] + props['min_ver'] + 'r' + props['rev']
	props['new_version'] = props['maj_ver'] + props['min_ver'] + 'r' + props['new_rev']
	return props
}

boolean NATIVES_CHANGED = false

pipeline {
	options {
		skipDefaultCheckout() // Specialiced checkout is performed below
		timestamps()
		timeout(time: 180, unit: 'MINUTES')
		buildDiscarder(logRotator(numToKeepStr:'5'))
		disableConcurrentBuilds(abortPrevious: true)
	}
	agent {
		label 'centos-latest'
	}
	tools {
		jdk 'openjdk-jdk17-latest'
		maven 'apache-maven-latest'
	}
	environment {
		MAVEN_OPTS = "-Xmx4G"
		PR_VALIDATION_BUILD = "true"
	}
	parameters {
		booleanParam(name: 'forceNativeBuilds', defaultValue: false, description: 'Forces to run the native builds of swt\'s binaries. Will push the built binaries to the master branch, unless \'skipCommit\' is set. Useful in debugging.')
		booleanParam(name: 'skipCommit', defaultValue: false, description: 'Stops committing to swt and swt binaries repo at the end. Useful in debugging.')
	}
	stages {
		stage('Checkout swt git repos') {
			steps {
				dir('eclipse.platform.swt') {
					checkout scm
					script {
						def authorMail = sh(script: 'git log -1 --pretty=format:"%ce" HEAD', returnStdout: true)
						echo 'HEAD commit author: ' + authorMail
						if ('eclipse-releng-bot@eclipse.org'.equals(authorMail) && !params.forceNativeBuilds) {
							// Prevent endless build-loops due to self triggering because of a previous automated build of SWT-natives and the associated updates.
							currentBuild.result = 'ABORTED'
							error('Abort build only triggered by automated SWT-natives update.')
						}
					}
					sh '''
						git version
						git lfs version
						git config --unset core.hooksPath # Jenkins disables hooks by default as security feature, but we need the hooks for LFS
						git lfs update # Install Git LFS hooks in repository, which has been skipped due to the initially nulled hookspath
						git lfs pull
						git fetch --all --tags --quiet
						git remote set-url --push origin git@github.com:eclipse-platform/eclipse.platform.swt.git
					'''
				}
			}
		}
		stage('Check if SWT-binaries build is needed') {
			steps {
				dir('eclipse.platform.swt') {
					sh'''
						java -version
						git config --global user.email 'eclipse-releng-bot@eclipse.org'
						git config --global user.name 'Eclipse Releng Bot'
					'''
					script {
						def swtTag = getLatestGitTag()
						echo "Current tag=${swtTag}."
						boolean nativesChanged = false
						dir('bundles/org.eclipse.swt') {
							// Verify preprocessing is completed
							sh '''
								if grep -R --include='*.java' --line-number --fixed-strings -e 'int /*long*/' -e 'float /*double*/' -e 'int[] /*long[]*/' -e 'float[] /*double[]*/' .; then
									echo There are files with the wrong long /*int*/ preprocessing.
									exit 1
								fi
							'''
							def sourceFoldersProps = readProperties(file: 'nativeSourceFolders.properties')
							def sources = sourceFoldersProps.collectEntries{ k, src -> [ k, src.split(',').collect{ f -> '\'' + f + '\''}.join(' ') ] }
							def diff = sh(script: "git diff HEAD ${swtTag} ${sources.values().join(' ')}", returnStdout: true)
							nativesChanged = !diff.strip().isEmpty()
							echo "Natives changed since ${swtTag}: ${nativesChanged}"
						}
						if (nativesChanged || params.forceNativeBuilds) {
							NATIVES_CHANGED = true
							def swtVersions = getSWTVersions()
							withEnv(['swt_version='+swtVersions['swt_version'], 'new_version='+swtVersions['new_version'], 'rev='+swtVersions['rev'], 'new_rev='+swtVersions['new_rev'],
									'comma_ver='+swtVersions['comma_ver'], "new_comma_ver=${swtVersions['maj_ver']},${swtVersions['min_ver']},${swtVersions['new_rev']},0" ]) {
								sh '''
									# Delete native binaries to be replaced by subsequent binaries build
									rm binaries/org.eclipse.swt.gtk.*/libswt-*.so
									rm binaries/org.eclipse.swt.win32.*/swt-*.dll
									rm binaries/org.eclipse.swt.cocoa.*/libswt-*.jnilib
									
									echo "Incrementing version from ${swt_version} to ${new_version}; new comma_ver=${new_comma_ver}"
									
									libraryFile='bundles/org.eclipse.swt/Eclipse SWT PI/common/org/eclipse/swt/internal/Library.java'
									sed -i -e "s/REVISION = ${rev}/REVISION = ${new_rev}/g" "$libraryFile"
									
									commonMakeFile='bundles/org.eclipse.swt/Eclipse SWT/common/library/make_common.mak'
									sed -i -e "s/rev=${rev}/rev=${new_rev}/g" "$commonMakeFile"
									sed -i -e "s/comma_ver=${comma_ver}/comma_ver=${new_comma_ver}/g" "$commonMakeFile"
								'''
							}
						}
					}
				}
			}
		}
		stage('Build SWT-binaries, if needed') {
			when {
				expression { NATIVES_CHANGED }
			}
			matrix {
				axes {
					axis {
						name 'PLATFORM'
						values 'cocoa.macosx.aarch64' , 'cocoa.macosx.x86_64', 'gtk.linux.aarch64', 'gtk.linux.ppc64le', 'gtk.linux.x86_64', 'win32.win32.x86_64'
					}
				}
				stages {
					stage("Collect SWT-native's sources") {
						steps {
							dir('eclipse.platform.swt/bundles/org.eclipse.swt') {
								sh '''
									pfSpec=(${PLATFORM//"."/ })
									java -Dws=${pfSpec[0]} -Darch=${pfSpec[2]} build-scripts/CollectSources.java -nativeSources \
										"${WORKSPACE}/eclipse.platform.swt/binaries/org.eclipse.swt.${PLATFORM}/target/natives-build-temp"
								'''
							}
							dir("eclipse.platform.swt/binaries/org.eclipse.swt.${PLATFORM}/target/natives-build-temp") {
								stash(name:"swt.binaries.sources.${PLATFORM}")
							}
						}
					}
					stage('Build SWT-natives') {
						options {
							timeout(time: 120, unit: 'MINUTES') // Some build agents are rare and it might take awhile until they are available.
						}
						steps {
							script {
								def (ws, os, arch) = env.PLATFORM.split('\\.')
								dir("jdk-download-${os}.${arch}") {
									// Fetch the JDK, which provides the C header-files and shared native libaries, against which the natives are build.
									sh "curl ${getNativeJdkUrl(os, arch)} | tar -xzf - include/ lib/"
									stash name:"jdk.resources.${os}.${arch}", includes: "include/,lib/"
									deleteDir()
								}
								nativeBuildAgent("${PLATFORM}") {
									cleanWs() // Workspace is not cleaned up by default, so we do it explicitly
									echo "OS: ${os}, ARCH: ${arch}"
									unstash "swt.binaries.sources.${PLATFORM}"
									dir('jdk.resources') {
										unstash "jdk.resources.${os}.${arch}"
									}
									withEnv(['MODEL=' + arch, "OUTPUT_DIR=${WORKSPACE}/libs", "SWT_JAVA_HOME=${WORKSPACE}/jdk.resources"]) {
										if (isUnix()){
											sh '''
												mkdir libs
												sh build.sh install
												ls -1R libs
											'''
										} else {
											withEnv(['PATH=C:\\tools\\cygwin\\bin;' + env.PATH]) {
												bat '''
													mkdir libs
													cmd /c build.bat install
													ls -1R libs
												'''
											}
										}
									}
									dir('libs') {
										stash "swt.binaries.${PLATFORM}"
									}
								}
							}
						}
					}
					stage('Collect and sign binaries') {
						steps {
							dir("libs/${PLATFORM}") {
								unstash "swt.binaries.${PLATFORM}"
								sh '''
									if [[ ${PLATFORM} == cocoa.macosx.* ]]; then
										binariesExtension='jnilib'
										signerUrl='https://cbi.eclipse.org/macos/codesign/sign'
									elif [[ ${PLATFORM} == gtk.linux.* ]]; then
										binariesExtension='so'
									elif [[ ${PLATFORM} == win32.win32.* ]]; then
										binariesExtension='dll'
										signerUrl='https://cbi.eclipse.org/authenticode/sign'
									fi
									if [[ -n "$signerUrl" ]]; then
										echo "Sign ${PLATFORM} libraries"
										if [[ "${BRANCH_NAME}" == master ]] || [[ "${BRANCH_NAME}" =~ R[0-9]+_[0-9]+(_[0-9]+)?_maintenance ]]; then
											for file in *.${binariesExtension}; do
												mv $file unsigned-$file
												curl --fail --form "file=@unsigned-$file" --output "$file" "$signerUrl"
												rm unsigned-$file
											done
										fi
									fi
									cp *.$binariesExtension "${WORKSPACE}/eclipse.platform.swt/binaries/org.eclipse.swt.${PLATFORM}/"
								'''
							}
						}
					}
				}
			}
		}
		stage('Commit SWT-native binaries, if build') {
			when {
				expression { NATIVES_CHANGED }
			}
			steps {
				dir('eclipse.platform.swt') {
					withEnv(["swt_version=${getSWTVersions()['swt_version']}"]) { // versions are read from updated file
						sh '''
							find binaries -name "*${swt_version}*" -type f -exec chmod 755 {} +
							
							git add --all *
							git status
							git commit -m "v${swt_version}"
							git tag "v${swt_version}"
							
							git status
							git log --patch -2
						'''
					}
				}
			}
		}
		stage('Build') {
			steps {
				xvnc(useXauthority: true) {
					dir('eclipse.platform.swt') {
						sh '''
							mvn clean verify \
								--batch-mode --threads 1C -V -U -e -DforkCount=0 \
								-Papi-check \
								-Dcompare-version-with-baselines.skip=false \
								-Dorg.eclipse.swt.tests.junit.disable.test_isLocal=true \
								-Dmaven.test.failure.ignore=true -Dmaven.test.error.ignore=true
						'''
					}
				}
			}
			post {
				always {
					junit 'eclipse.platform.swt/tests/*.test*/target/surefire-reports/*.xml'
					archiveArtifacts artifacts: '**/*.log,**/*.html,**/target/*.jar,**/target/*.zip'
					discoverGitReferenceBuild referenceJob: 'eclipse.platform.swt/master'
					recordIssues publishAllIssues: true, tools: [eclipse(name: 'Compiler and API Tools', pattern: '**/target/compilelogs/*.xml'), javaDoc()], qualityGates: [[threshold: 1, type: 'DELTA', unstable: true]]
					recordIssues publishAllIssues: true, tool: mavenConsole(), qualityGates: [[threshold: 1, type: 'DELTA_ERROR', unstable: true]]
				}
			}
		}
		stage('Push SWT-native binaries, if build') {
			when {
				expression { NATIVES_CHANGED }
			}
			steps {
				sshagent(['github-bot-ssh']) {
					dir('eclipse.platform.swt') {
						sh """
							# Check for the master-branch as late as possible to have as much of the same behaviour as possible
							if [[ '${BRANCH_NAME}' == master ]] || [[ '${BRANCH_NAME}' =~ R[0-9]+_[0-9]+(_[0-9]+)?_maintenance ]]; then
								if [[ ${params.skipCommit} != true ]]; then
									
									# Don't rebase and just fail in case another commit has been pushed to the master/maintanance branch in the meantime
									git push origin HEAD:refs/heads/${BRANCH_NAME}
									git push origin refs/tags/${getLatestGitTag()}
									
									exit 0
								else
									echo Committing is skipped
								fi
							else
								echo Skip pushing changes of native-binaries for branch '${BRANCH_NAME}'
							fi
						"""
					}
				}
			}
		}
	}
}
