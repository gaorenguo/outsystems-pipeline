pipeline {
  agent none // Each pipeline stage must specify which local agent will run the stage
  parameters {
    // App List Parameters -> automatically filled by LT Trigger plugin
    string(name: 'ApplicationScope', defaultValue: '', description: 'Comma-separated list of LifeTime applications to deploy.')
    string(name: 'ApplicationScopeWithTests', defaultValue: '', description: 'Comma-separated list of LifeTime applications to deploy (including test applications)')
    string(name: 'TriggeredBy', defaultValue: 'N/A', description: 'Name of LifeTime user that triggered the pipeline remotely.')
  }
  options { skipStagesAfterUnstable() }
  environment {
    // Artifacts Folder
    ArtifactsFolder = "Artifacts"
    // LifeTime Specific Variables
    LifeTimeHostname = 'pcjcl-lt.outsystemsenterprise.com'
    LifeTimeAPIVersion = 2
    // Authentication Specific Variables
    AuthorizationToken = credentials('LifeTimeServiceAccountToken')
    // Environments Specification Variables
    /*
    * Pipeline for 5 Environments:
    * DevelopmentEnvironment -> Where you develop your applications. This should be the default environment you connect with service studio.
    * RegressionEnvironment -> Where your automated tests will test your applications.
    * AcceptanceEnvironment -> Where you run your acceptance tests of your applications.
    * PreProductionEnvironment -> Where you prepare your apps to go live.
    * ProductionEnvironment -> Where your apps are live.
    */
    DevelopmentEnvironment = 'development environment # 3'
    RegressionEnvironment = 'non-production environment # 2'
    AcceptanceEnvironment = 'non-production environment # 2'
    PreProductionEnvironment = 'production environment # 1'
    ProductionEnvironment = 'production environment # 1'
    // Regression URL Specification
    ProbeEnvironmentURL = 'https://pcjcl-dev.outsystemsenterprise.com/'
    BddEnvironmentURL = 'https://pcjcl-dev.outsystemsenterprise.com/'
    // OutSystems PyPI package version
    OSPackageVersion = '0.3.1'
  }
  stages {
    stage('Get and Deploy Latest Tags') {
      agent any // Replace by specific label for narrowing down to OutSystems pipeline-specific agents
      steps {
        echo "Pipeline run triggered remotely by '${params.TriggeredBy}' for the following applications (including tests): '${params.ApplicationScopeWithTests}'"
        echo "Create ${env.ArtifactsFolder} Folder"
        // Create folder for storing artifacts
        powershell "mkdir ${env.ArtifactsFolder}"
        // Only the virtual environment needs to be installed at the system level
        echo "Install Python Virtual environments"
        powershell 'C:\\Users\\P0168716\\AppData\\Local\\Programs\\Python\\Python37\\Scripts\\pip install -q -I virtualenv --user'
        withPythonEnv('C:\\Users\\P0168716\\AppData\\Local\\Programs\\Python\\Python37\\python') {
          echo "Install Python requirements"
          // Install the rest of the dependencies at the environment level and not the system level
          powershell "pip install -U outsystems-pipeline==\"${env.OSPackageVersion}\""
          echo 'Retrieving latest application tags from Development environment...'
          // Retrive the Applications and Environment details from the Source environment
          powershell "C:\\Users\\P0168716\\AppData\\Local\\Programs\\Python\\Python37\\python -m outsystems.pipeline.fetch_lifetime_data --artifacts \"${env.ArtifactsFolder}\" --lt_url ${env.LifeTimeHostname} --lt_token ${env.AuthorizationToken} --lt_api_version ${env.LifeTimeAPIVersion}"
          echo 'Deploying latest application tags to Regression...'
          // Deploy the application list, with tests, to the Regression environment
          lock('deployment-plan-REG') {
            powershell "C:\\Users\\P0168716\\AppData\\Local\\Programs\\Python\\Python37\\python -m outsystems.pipeline.deploy_latest_tags_to_target_env --artifacts \"${env.ArtifactsFolder}\" --lt_url ${env.LifeTimeHostname} --lt_token ${env.AuthorizationToken} --lt_api_version ${env.LifeTimeAPIVersion} --source_env \"${env.DevelopmentEnvironment}\" --destination_env \"${env.RegressionEnvironment}\" --app_list \"${params.ApplicationScopeWithTests}\""
          }
        }
      }
      post {
        // It will always store the cache files generated, for observability purposes
        always {
          dir ("${env.ArtifactsFolder}") {
            archiveArtifacts artifacts: "*.cache", onlyIfSuccessful: true
            archiveArtifacts artifacts: "*_data/*.cache", onlyIfSuccessful: true
            stash name: "manifest",includes: "deployment_data/deployment_manifest.cache", allowEmpty: true
          }
        }
        // If there's a failure, tries to store the Deployment conflicts (if exists), for observability and troubleshooting purposes
        failure {
          dir ("${env.ArtifactsFolder}") {
            archiveArtifacts artifacts: "DeploymentConflicts"
          }
        }
        // Delete artifacts folder to avoid impacts in subsquent builds
        cleanup {
          dir ("${env.ArtifactsFolder}") {
            deleteDir()
          }
        }
      }
    }
    stage('Run Regression') {
      when {
        expression { return params.ApplicationScope != params.ApplicationScopeWithTests }
      }
      agent any // Replace by specific label for narrowing down to OutSystems pipeline-specific agents
      steps {
        echo "Create ${env.ArtifactsFolder} Folder"
        // Create folder for storing artifacts
        powershell "mkdir ${env.ArtifactsFolder}"
        // Only the virtual environment needs to be installed at the system level
        echo "Install Python Virtual environments"
        powershell 'pip install -q -I virtualenv --user'
        withPythonEnv('python') {
          echo "Install Python requirements"
          // Install the rest of the dependencies at the environment level and not the system level
          powershell "pip install -U outsystems-pipeline==\"${env.OSPackageVersion}\""
          echo 'Generating URLs for BDD testing...'
          // Generate the URL endpoints of the BDD tests
          powershell "python -m outsystems.pipeline.generate_unit_testing_assembly --artifacts \"${env.ArtifactsFolder}\" --app_list \"${params.ApplicationScopeWithTests}\" --cicd_probe_env ${env.ProbeEnvironmentURL} --bdd_framework_env ${env.BddEnvironmentURL}"
          echo "Testing the URLs and generating the JUnit results XML..."
          // Run those tests and generate a JUNIT test report
          powershell(script: "python -m outsystems.pipeline.evaluate_test_results --artifacts \"${env.ArtifactsFolder}\"", returnStatus: true)
        }
      }
      post {
        always {
          withPythonEnv('python') {
            echo "Publishing JUnit test results..."
            junit(testResults: "${env.ArtifactsFolder}\\junit-result.xml", allowEmptyResults: true)
          }
          dir ("${env.ArtifactsFolder}") {
            archiveArtifacts artifacts: "*_data/*.cache", onlyIfSuccessful: true
          }
        }
        cleanup {
          dir ("${env.ArtifactsFolder}") {
            deleteDir()
          }
        }
      }
    }
    /**
    stage('Deploy Acceptance') {
      agent any // Replace by specific label for narrowing down to OutSystems pipeline-specific agents
      steps {
        echo "Create ${env.ArtifactsFolder} Folder"
        // Create folder for storing artifacts
        powershell "mkdir ${env.ArtifactsFolder}"
        // Unstash Deployment Manifest file to mantain consistency throughout the pipeline execution
        dir ("${env.ArtifactsFolder}") {
            unstash "manifest"
        }
        // Only the virtual environment needs to be installed at the system level
        echo "Install Python Virtual environments"
        powershell 'pip install -q -I virtualenv --user'
        withPythonEnv('python') {
          echo "Install Python requirements"
          // Install the rest of the dependencies at the environment level and not the system level
          powershell "pip install -U outsystems-pipeline==\"${env.OSPackageVersion}\""
          echo 'Deploying latest application tags to Acceptance...'
          // Deploy the application list, without tests, to the Acceptance environment
          lock('deployment-plan-ACC') {
            powershell "python -m outsystems.pipeline.deploy_latest_tags_to_target_env --artifacts \"${env.ArtifactsFolder}\" --lt_url ${env.LifeTimeHostname} --lt_token ${env.AuthorizationToken} --lt_api_version ${env.LifeTimeAPIVersion} --source_env \"${env.RegressionEnvironment}\" --destination_env \"${env.AcceptanceEnvironment}\" --app_list \"${params.ApplicationScope}\" --manifest \"${env.ArtifactsFolder}\\deployment_data\\deployment_manifest.cache\""
          }
        }
      }
      post {
        always {
          dir ("${env.ArtifactsFolder}") {
            archiveArtifacts artifacts: "*_data/*.cache", onlyIfSuccessful: true
          }
        }
        failure {
          dir ("${env.ArtifactsFolder}") {
            archiveArtifacts artifacts: "DeploymentConflicts"
          }
        }
        cleanup {
          dir ("${env.ArtifactsFolder}") {
            deleteDir()
          }
        }
      }
    }
    stage('Accept Changes') {
      agent none // Approval gates do not require allocating an agent from the pool
      steps {
        // Define milestone before approval gate to manage concurrent builds
        milestone(ordinal: 40, label: 'before-approval')
        // Wrap the confirm option in a timeout to avoid hanging Jenkins forever
        timeout(time:1, unit:'DAYS') {
          input 'Accept changes and deploy to Production?'
        }
        // Discard previous builds that have not been accepted yet
        milestone(ordinal: 50, label: 'after-approval')
      }
    }
    stage('Deploy Dry-Run') {
      agent any // Replace by specific label for narrowing down to OutSystems pipeline-specific agents
      steps {
        echo "Create ${env.ArtifactsFolder} Folder"
        // Create folder for storing artifacts
        powershell "mkdir ${env.ArtifactsFolder}"
        // Unstash Deployment Manifest file to mantain consistency throughout the pipeline execution
        dir ("${env.ArtifactsFolder}") {
            unstash "manifest"
        }
        // Only the virtual environment needs to be installed at the system level
        echo "Install Python Virtual environments"
        powershell 'pip install -q -I virtualenv --user'
        withPythonEnv('python') {
          echo "Install Python requirements"
          // Install the rest of the dependencies at the environment level and not the system level
          powershell "pip install -U outsystems-pipeline==\"${env.OSPackageVersion}\""
          echo 'Deploying latest application tags to Pre-Production...'
          // Deploy the application list, without tests, to the Pre-Production environment
          lock('deployment-plan-PRE') {
            powershell "python -m outsystems.pipeline.deploy_latest_tags_to_target_env --artifacts \"${env.ArtifactsFolder}\" --lt_url ${env.LifeTimeHostname} --lt_token ${env.AuthorizationToken} --lt_api_version ${env.LifeTimeAPIVersion} --source_env \"${env.AcceptanceEnvironment}\" --destination_env \"${env.PreProductionEnvironment}\" --app_list \"${params.ApplicationScope}\" --manifest \"${env.ArtifactsFolder}\\deployment_data\\deployment_manifest.cache\""
          }
        }
      }
      post {
        always {
          dir ("${env.ArtifactsFolder}") {
            archiveArtifacts artifacts: "*_data/*.cache", onlyIfSuccessful: true
          }
        }
        failure {
          dir ("${env.ArtifactsFolder}") {
            archiveArtifacts artifacts: "DeploymentConflicts"
          }
        }
        cleanup {
          dir ("${env.ArtifactsFolder}") {
            deleteDir()
          }
        }
      }
    }
    stage('Deploy Production') {
      agent any // Replace by specific label for narrowing down to OutSystems pipeline-specific agents
      steps {
        echo "Create ${env.ArtifactsFolder} Folder"
        // Create folder for storing artifacts
        powershell "mkdir ${env.ArtifactsFolder}"
        // Unstash Deployment Manifest file to mantain consistency throughout the pipeline execution
        dir ("${env.ArtifactsFolder}") {
            unstash "manifest"
        }
        // Only the virtual environment needs to be installed at the system level
        echo "Install Python Virtual environments"
        powershell 'pip install -q -I virtualenv --user'
        withPythonEnv('python') {
          echo "Install Python requirements"
          // Install the rest of the dependencies at the environment level and not the system level
          powershell "pip install -U outsystems-pipeline==\"${env.OSPackageVersion}\""
          echo 'Deploying latest application tags to Production...'
          // Deploy the application list, without tests, to the Production environment
          lock('deployment-plan-PRD') {
            powershell "python -m outsystems.pipeline.deploy_latest_tags_to_target_env --artifacts \"${env.ArtifactsFolder}\" --lt_url ${env.LifeTimeHostname} --lt_token ${env.AuthorizationToken} --lt_api_version ${env.LifeTimeAPIVersion} --source_env \"${env.PreProductionEnvironment}\" --destination_env \"${env.ProductionEnvironment}\" --app_list \"${params.ApplicationScope}\" --manifest \"${env.ArtifactsFolder}\\deployment_data\\deployment_manifest.cache\""
          }
        }
      }
      post {
        always {
          dir ("${env.ArtifactsFolder}") {
            archiveArtifacts artifacts: "*_data/*.cache", onlyIfSuccessful: true
          }
        }
        failure {
          dir ("${env.ArtifactsFolder}") {
            archiveArtifacts artifacts: "DeploymentConflicts"
          }
        }
        cleanup {
          dir ("${env.ArtifactsFolder}") {
            deleteDir()
          }
        }
      }
    }
    */
  }
}
