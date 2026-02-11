def evalValue(expression, path = null) {
	return sh( returnStdout: true,
		script: "mvn org.apache.maven.plugins:maven-help-plugin:evaluate -Dexpression=${ expression } -q -DforceStdout ${ path==null ? '' : ('-pl='+path) } | tail -1")
}

def installArtifact(mod, parent = null) {
	if(!mod.active()) {
		skipStage()
		return
	}
	def relPath = (parent == null ? null : mod.relPathFrom(parent))
	// get module metadata
	def groupId = evalValue('project.groupId', relPath)
	def artifactId = evalValue('project.artifactId', relPath)
	def version = evalValue('project.version', relPath)
	echo "Building: ${ groupId }:${ artifactId }:${ version }"
	try {
		sh "mvn-dev -P ${ REPOS },toolchain-openjdk-1-8-0,ci-install ${ relPath==null ? '' : ('-pl='+relPath) }"
	} finally {
		def baseName = "${ artifactId }-${ version }"
		// create spec .pom in target/ path
		sh "cp -T '${ mod.path() }/pom.xml' '${ mod.path() }/target/${ baseName }.pom'"
		// archive artifacts
		dir(path: "${ mod.path() }/target") {
			sh 'ls -l'
			archiveArtifacts artifacts: "${ baseName }.pom", fingerprint: true
			if(mod.hasTag('pack-jar')) {
				archiveArtifacts artifacts: "${ baseName }*.jar", fingerprint: true
			}
		}
		// create signatures
		signArtifacts(artifacts: "${ baseName }*")
		// bundle artifacts + signatures
		bundleArtifacts( bundle: mod.id(), artifacts: "${ baseName }.pom*", metadata: [
			'groupId': groupId, 'artifactId': artifactId, 'version': version
		])
		for (test in [ false, true ]) {
			for (classifier in [ '', 'javadoc', 'sources' ]) {
				if(test)
					classifier = classifier=='' ? 'tests' : ('test-'+classifier)
				bundleArtifacts( bundle: mod.id(), artifacts: "${ baseName }${ classifier=='' ? '' : ('-'+classifier) }.jar*", metadata: [
					'groupId': groupId, 'artifactId': artifactId, 'version': version, 'classifier': classifier
				])
			}
		}
	}
}

node( label: 'linux' ) {
	withModules {
		tool(name: 'maven-latest', type: 'maven')

		stage('Checkout SCM') {
			checkout(scm)
		}

		sh 'chmod +x $WORKSPACE/.build/*'
		env.setProperty('PATH+SCRIPTS', "${ env.WORKSPACE }/.build")
		env.GLOBAL_MAVEN_SETTINGS     = '/srv/jenkins/.m2/global-settings.xml'
		env.MAVEN_SETTINGS            = "${ env.WORKSPACE }/.mvn/settings.xml"
		env.MAVEN_TOOLCHAINS          = "${ env.WORKSPACE }/.mvn/toolchains.xml"
		if(env.BRANCH_NAME == 'master') {
			env.REPOS = 'repo-releases'
		} else {
			env.REPOS = 'repo-releases,repo-development'
		}

		stage('Initialize') {
			env.RESULT_PATH  = "${ WORKSPACE }/result/"
			env.ARCHIVE_PATH = "${ WORKSPACE }/archive/"
			sh "mkdir -p ${ RESULT_PATH }"
			sh "mkdir -p ${ ARCHIVE_PATH }"
			
			addModule( id: 'maven-pipeline-workflow',  path: '.',  name: 'Maven Pipeline Workflow',  tags: [ /*'test',*/ 'pack-jar' ] )
		}

		stage('Init Modules') {
			sshagent (credentials: ['RunedUniverse-Jenkins']) {
				perModule(failFast: true) {
					def mod = getModule();
					mod.activate(
						!mod.hasTag('skip') && sh(
								returnStdout: true,
								script: "git-check-version-tag ${ mod.id() } ."
							) == '1'
					);
				}
			}
		}
		stage ('Info') {
			sh 'printenv | sort'
		}

		stage('Update Maven Repo') {
			if(checkAllModules(match: 'all', active: false)) {
				skipStage()
				return
			}
			sh "mvn-dev -P ${ REPOS } dependency:purge-local-repository -DactTransitively=false -DreResolve=false"
			sh "mvn-dev -P ${ REPOS },ci-install,ci-validate dependency:go-offline -U --fail-never"
		}

		stage('Code Validation') {
			sh "mvn-dev -P ${ REPOS },ci-validate --fail-at-end -T1C"
		}

		bundleContext {
			stage('Install Workflow Plugin') {
				installArtifact( getModule(id: 'maven-pipeline-workflow') );
			}

			stage('Test') {
				if(!checkAllModules(withTagIn: [ 'test' ], active: true)) {
					skipStage()
					return
				}
				sh "mvn-dev -P ${ REPOS },toolchain-openjdk-1-8-0,ci-test-build"
				sh "mvn-dev --fail-never -P ${ REPOS },toolchain-openjdk-1-8-0,ci-test-exec,test-system"
				// check tests, archive reports in case junit flags errors
				junit '*/target/surefire-reports/*.xml'
				if(currentBuild.resultIsWorseOrEqualTo('UNSTABLE')) {
					archiveArtifacts artifacts: '*/target/surefire-reports/*.xml'
				}
			}

			stage('Package Build Result') {
				if(checkAllModules(match: 'all', active: false)) {
					skipStage()
					return
				}
				dir(path: "${ env.RESULT_PATH }") {
					unarchive mapping: ['*':'.']
					sh 'ls -l'
					sh "tar -I \"pxz -9\" -cvf ${ ARCHIVE_PATH }artifacts.tar.xz *"
					sh "zip -9 ${ ARCHIVE_PATH }artifacts.zip *"
				}
				dir(path: "${ env.ARCHIVE_PATH }") {
					archiveArtifacts artifacts: '*', fingerprint: true
				}
			}

			stage('Deploy') {
				perModule {
					def mod = getModule();
					if(!mod.active()) {
						skipStage()
						return
					}
					// bundle info
					bundleInfo( bundle: mod.id(), metadata: true )
					// deploy to development repo
					stage('Develop'){
						deployArtifacts( bundle: mod.id(), repo: 'nexus-runeduniverse>maven-development' )
					}
					// deploy to release repo
					stage('Release') {
						if(currentBuild.resultIsWorseOrEqualTo('UNSTABLE') || env.BRANCH_NAME != 'master') {
							skipStage()
							return
						}
						deployArtifacts( bundle: mod.id(), repo: 'nexus-runeduniverse>maven-releases' )
						def groupId = evalValue('project.groupId')
						def artifactId = evalValue('project.artifactId')
						def version = evalValue('project.version')
						sshagent (credentials: ['RunedUniverse-Jenkins']) {
							sh "git tag -a ${ mod.id() }/v${ version } -f -m '[artifact] ${ groupId }:${ artifactId }:${ version }'"
							sh "git push origin ${ mod.id() }/v${ version }"
						}
					}
					// merge bundles into default
					bundleMerge( source: mod.id() )
				}
				stage('Stage at Maven-Central') {
					if(currentBuild.resultIsWorseOrEqualTo('UNSTABLE') || env.BRANCH_NAME != 'master') {
						skipStage()
						return
					}
					deployArtifacts( repo: 'maven-central>net.runeduniverse' )
				}
			}
		}

		cleanWs()
	}
}
