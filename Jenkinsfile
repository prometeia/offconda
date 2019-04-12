@Library(['promebuilder', 'offconda'])_


pipeline {
  agent any
  parameters {
    string(
        name: 'COMPONENTS',
        description: 'Final packages and version (e.g. "pytho=4.6.4 gsf=4.6.6rc2 ratingpro=3.6.5 pytho_docs=4.6.3 serversoa=1.0.5 python=2.7.15 conda=4.6 conda-env=2.6")',
        defaultValue: 'pytho=4.6.4 gsf=4.6.7 ratingpro=3.6.6 pytho_docs=4.6.3 serversoa=1.0.5 python=2.7.15 conda=4.6 conda-env=2.6'
    )
    string(
        name: 'LABEL',
        defaultValue: env.TAG_NAME ? (env.TAG_NAME.contains('rc') ? 'release' : 'main') : env.BRANCH_NAME.split('/')[0].replace('hotfix', 'release').replace('master', 'main'),
        description: 'Source label'
    )
    string(
        name: 'TARGET',
        defaultValue: 'C:\\CONDAOFFLINE',
        description: 'Target offline repository'
    )
  }
  environment {
    CONDAENV = "${env.JOB_NAME}_${env.BUILD_NUMBER}".replace('%2F','_').replace('/', '_')
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
            doublePackager('linux', params.LABEL, params.COMPONENTS + " supervisor==3.* libgcc-ng==7.2.* libstdcxx-ng=7.2.*")
          }
        }
      }
    }
    stage('Downloading and Indexing') {
      when {
        buildingTag()
      }
      steps {
        unarchive(mapping: ["elencone-linux.txt": "elencone-linux.txt"])
        bat(script: "python download.py ${env.TAG_NAME}")
        bat(script: "call conda index ${env.TAG_NAME}")
        // Solo indici, please!
        // archiveArtifacts artifacts: "${env.TAG_NAME}/*/*.tag.bz2"
      }
    }
    stage('Checking') {
      when {
        buildingTag()
      }
      steps {
        bat(script: "python distrocheck.py ${env.TAG_NAME}")
      }
    }
    stage('Publishing') {
      when {
        buildingTag()
      }
      steps {
        bat(script: "(robocopy /MIR ${env.TAG_NAME} ${params.TARGET}\\${env.TAG_NAME} /XD ${env.TAG_NAME}\\linux-64\\.cache ${env.TAG_NAME}\\noarch\\.cache ) ^& IF %ERRORLEVEL% LEQ 1 exit 0")
      }
    }
    stage('Testing') {
      when {
        buildingTag()
      }
      steps {
        node('linux') {
          sh(script: "conda install pytho ratingpro serversoa -c http://daa-ws-01:9200/.condaoffline/${env.TAG_NAME} --override-channels --dry-run")
        }
      }
    }
  }
  post {
    success {
      slackSend color: "good", message: "Successed ${env.JOB_NAME} (<${env.BUILD_URL}|Open>)"
      deleteDir()
    }
    failure {
      slackSend color: "warning", message: "Failed ${env.JOB_NAME} (<${env.BUILD_URL}|Open>)"
    }
  }
}
