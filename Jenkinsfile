@Library(['promebuilder', 'offconda'])_


pipeline {
  agent any
  parameters {
    string(
        name: 'QUARTER',
        description: 'Version name of the whole solution (e.g. "PYTHO_07")',
        defaultValue: env.TAG_NAME
    )
    string(
        name: 'COMPONENTS',
        description: 'Final packages and their version (e.g. "pytho==4.3.* gsf==4.3.* ratingpro==3.4.0 serversoa==1.0.* pytho_docs==4.3.* conda python==2.7.*")',
        defaultValue: 'pytho==4.6.* gsf==4.6.* ratingpro==3.6.* serversoa==1.0.* pytho_docs==4.6.* python==2.7.15 conda==4.6.*'
    )
    string(
        name: 'TARGET',
        defaultValue: 'C:\\CONDAOFFLINE',
        description: 'Target offline repository'
    )
  }
  environment {
    CONDAENV = "${env.JOB_NAME}_${env.BUILD_NUMBER}".replace('%2F','_').replace('/', '_')
    LABEL = env.BRANCH_NAME.split('/')[0].replace('hotfix', 'release').replace('master', 'main')
  }
  stages {
    stage('Bootstrap') {
      steps {
        echo "NB: The packages should be PRIVATE o PUBLIC, it doesn't work with 'authentication required'."
        writeFile file: 'components.txt', text: "${env.COMPONENTS}"
        archiveArtifacts artifacts: "components.txt"
        stash(name: "source", useDefaultExcludes: true)
      }
    }
    stage("PackageList") {
      parallel {
        stage("Build on Linux") {
          steps {
            doublePackager('linux', env.LABEL, params.COMPONENTS + " supervisor==3.*")
          }
        }
        stage("Build on Windows") {
          steps {
            doublePackager('windows', env.LABEL, params.COMPONENTS)
          }
        }
      }
    }
    stage('Downloading and Indexing') {
      when {
        buildingTag()
      }
      steps {
        unarchive(mapping: ["elencone-linux.txt": "elencone-linux.txt", "elencone-windows.txt": "elencone-windows.txt"])
        bat(script: "python download.py ${params.QUARTER}")
        bat(script: "call conda index ${params.QUARTER}")
        // Solo indici, please!
        // archiveArtifacts artifacts: "${params.QUARTER}/*/*.tag.bz2"
      }
    stage('Publishing') {
      when {
        buildingTag()
      }
      steps {
        bat(script: "(robocopy /MIR ${params.QUARTER} ${params.TARGET}\\${params.QUARTER}) ^& IF %ERRORLEVEL% LEQ 1 exit 0")
      }
    }
  }
  post {
    always {
      deleteDir()
    }
    success {
      slackSend color: "good", message: "Successed ${env.JOB_NAME} (<${env.BUILD_URL}|Open>)"
    }
    failure {
      slackSend color: "warning", message: "Failed ${env.JOB_NAME} (<${env.BUILD_URL}|Open>)"
    }
  }
}
