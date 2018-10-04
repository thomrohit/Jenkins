#!groovy


node {
	println "/***********************************************************/"
	println env.BRANCH_NAME
	println "/***********************************************************/"
		
	env.JAVA_HOME="${tool 'JDK8'}"
    env.PATH="${env.JAVA_HOME}/bin:${env.PATH}"
	
	// Define variables
	def TRANSFER_DIR='/home/userName/deploy'
	def PRIVATE_KEY='/home/jenkins/.ssh/id_rsa'
	def SERVER ='userName@ipAddress'
	
	// pull request or feature branch
    if  (env.BRANCH_NAME != 'master') {
        checkout()
		up2dateCode()
        build()
        unitTest()
       // test whether this is a regular branch build or a merged PR build
        if (!isPRMergeBuild()) {
           allCodeQualityTests()
       } else {
            // Pull request
	    deploy(TRANSFER_DIR, PRIVATE_KEY, SERVER)
        serverUp()
        } 
    } // master branch / production
    else { 
        checkout()
        build()
        unitTest()
        allCodeQualityTests()
        production()
    }
	}
	
	def isPRMergeBuild() {
    return (env.BRANCH_NAME ==~ /^PR-\d+$/)
}

def checkout () {
    stage ('Checkout code') {
    context="continuous-integration/jenkins/"
    context += isPRMergeBuild()?"pr-merge/checkout":"branch/checkout"
    checkout scm
    setBuildStatus ("${context}", 'Check out completed', 'SUCCESS')
	}
}

def up2dateCode () {
	  stage ('Check Updated code') {
	  context="continuous-integration/jenkins/"
   	 context += isPRMergeBuild()?"pr-merge/checkout":"branch/updatedCode"
		// Below piece of code adds a remote(Git repo url) if it does'nt exist 
		sh '''  if git config remote.original.url > /dev/null; then
		echo "remote already present"
		else
		echo "adding orignal/upstream remote"
		git remote add original git@github.com:Repo.git
		fi '''
		
		// Fetch master code for SHA- commit ID
			sh "git fetch original master" 

		// Below piece of code compares the SHA of the acutal/original branch and SHA from which 
		// your Forked code split i.e. the common ancestor/parent from the spilt 
		// git merge-base– is a simple built-in utility to tell you where two branches diverged. It will tell you the commit where the two branches split off
		sh '''if [ $(git rev-parse original/master) == $(git merge-base original/master remotes/origin/'''+env.BRANCH_NAME+''') ];
		then
		echo "Your branch is up to date."
			exit 0
		else
		echo "You need to merge / rebase."
			exit 1
		fi''' 
		
		setBuildStatus ("${context}", 'Fork up to date', 'SUCCESS')
	  }
}

def build () {
    stage ('Build') {
    		//sh "mvn  --settings $mvnSettings clean  install site -DskipTests=true -Dadditionalparam=-Xdoclint:none"
		  mvn 'clean install -DskipTests=true -Dadditionalparam=-Xdoclint:none'
			
	}
}

def unitTest() {
    stage ('Unit tests') {
     context="continuous-integration/jenkins/"
   	 context += isPRMergeBuild()?"pr-merge/checkout":"branch/UnitTest"
    	 mvn 'test -B'
	
    	  step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
	     if (currentBuild.result == "UNSTABLE") {
	    // setBuildStatus ("${context}", 'Unit Test', 'FAILURE')
	      setBuildStatus ("${context}", 'Unit Test', 'UNSTABLE')
		//sh "exit 1"
	    }
    }
}


def allCodeQualityTests() {
    stage 'Code Quality'
    lintTest()
    coverageTest()
}

def lintTest() {
    context="continuous-integration/jenkins/linting"
    setBuildStatus ("${context}", 'Checking code conventions', 'PENDING')
    lintTestPass = true

    try {
        mvn 'verify -DskipTests=true'
    } catch (err) {
        setBuildStatus ("${context}", 'Some code conventions are broken', 'FAILURE')
        lintTestPass = false
    } finally {
        if (lintTestPass) setBuildStatus ("${context}", 'Code conventions OK', 'SUCCESS')
    }
}

def coverageTest() {
    context="continuous-integration/jenkins/coverage"
    setBuildStatus ("${context}", 'Checking code coverage levels', 'PENDING')

    coverageTestStatus = true

    try {
      //  mvn 'cobertura:check'
      checkstyle canComputeNew: false, defaultEncoding: '', healthy: '', pattern: '', unHealthy: ''
	findbugs canComputeNew: false, defaultEncoding: '', excludePattern: '', healthy: '', includePattern: '', pattern: '', unHealthy: ''
	pmd canComputeNew: false, defaultEncoding: '', healthy: '', pattern: '', unHealthy: ''
    } catch (err) {
        setBuildStatus("${context}", 'Code coverage below 90%', 'FAILURE')
        throw err
    }

    setBuildStatus ("${context}", 'Code coverage above 90%', 'SUCCESS')

}


def deploy(TRANSFER_DIR, PRIVATE_KEY, SERVER) {
	//replace project with your projectName
stage('Deploy') {
		echo " Deploy to Server"
			sh "ssh -i ${PRIVATE_KEY} ${SERVER} mkdir -p ${TRANSFER_DIR}"
			sh "scp -r -i ${PRIVATE_KEY} target/project-*-resources-dev.zip target/project-*.war ${SERVER}:${TRANSFER_DIR}"
			sh "ssh -i ${PRIVATE_KEY} ${SERVER} /home/userName/deploy.sh"
		}
}

//Check if the server is up and running by doing a DOS attack until a 200 response is achieved 
def serverUp () {
stage('ServerUp') {
	 sh "while [[ \$(curl --output /home/jenkins/output.log --silent -w '%{http_code}\n' --header 'Content-Type: text/xml;charset=UTF-8' --data '@/home/jenkins/request.xml' 'URL') != "200" ]] ; do sleep 10 ; done"
	 sh "/home/jenkins/serverUp.sh URL/IP"
	}
}

def production() {
    stage ('Deploy to Production') {
	//step([$class: 'ArtifactArchiver', artifacts: '**/target/*.jar', fingerprint: true])
    parallelDeploy()
	def version = 2.50 // read from pom.xml
    echo "Release version: ${version}"
    createRelease(version, new Date().format( 'dd-MM-yyyy' ))
    
	}
   
}

def parallelDeploy(){
	parallel (
            "PROD1" : {
                echo "deployed to PROD1"
				deploy(TRANSFER_DIR, PRIVATE_KEY_1, SERVER_1)
            },
            "PROD2" : {
                echo "deployed to PROD2"
				deploy(TRANSFER_DIR, PRIVATE_KEY_2, SERVER_2)
            }
        )
}

def mvn(args) {
    withMaven(
        // Maven installation declared in the Jenkins "Global Tool Configuration"
        maven: 'Maven',
        // Using global defined settings.xml file from Global Tool Configuration 
        // settings.xml referencing the PATH to use a different settings.xml
         //mavenSettingsFilePath: '/home/jenkins/.m2/settings.xml',
        // we do not need to set a special local maven repo, take the one from the standard box
        //mavenLocalRepo: '.repository'
        ) {
        // Run the maven build
        sh "mvn $args -Dmaven.test.failure.ignore"
    }
}

void createRelease(tagName, createdAt) {
def GITHUB_TOKEN ='8c***d6bf-**********-e22267***'
    withCredentials([[$class: 'StringBinding', credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_TOKEN']]) {
        def body = "**Created at:** ${createdAt}\n**Deployment job:** [${env.BUILD_NUMBER}](${env.BUILD_URL})\n**Environment:**)"
        def payload = JsonOutput.toJson(["tag_name": "v${tagName}", "name": "${env.HEROKU_PRODUCTION} - v${tagName}", "body": "${body}"])
        def apiUrl = "REPO_URL.git/releases"
	println body
	println payload
	
        def response = sh(returnStdout: true, script: "curl -s -H \"Authorization: Token ${env.GITHUB_TOKEN}\" -H \"Accept: application/json\" -H \"Content-type: application/json\" -X POST -d '${payload}' ${apiUrl}").trim()
    }
}

// Set the status as pass/fail on github side
void setBuildStatus(context, message, state) {
    step([
      $class: "GitHubCommitStatusSetter",
      contextSource: [$class: "ManuallyEnteredCommitContextSource", context: context],
      reposSource: [$class: "ManuallyEnteredRepositorySource", url: "REPO_URL.git"],
      errorHandlers: [[$class: "ChangingBuildStatusErrorHandler", result: "UNSTABLE"]],
      statusResultSource: [ $class: "ConditionalStatusResultSource", results: [[$class: "AnyBuildResult", message: message, state: state]] ]
  ]);
}