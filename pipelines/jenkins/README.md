     __  __      _   _   _       ____ ___
    |  \/  | ___| |_| |_| | ___ / ___|_ _|
    | |\/| |/ _ \ __| __| |/ _ \ |    | |
    | |  | |  __/ |_| |_| |  __/ |___ | |
    |_|  |_|\___|\__|\__|_|\___|\____|___|
    MettleCI DevOps for DataStage
    (C) 2021-2022 Data Migrators

Your Jenkins CI template can be found in the following files:

- `Jenkinsfile-DevOps` which is a DevOps template pipeline, and
- `Jenkinsfile-Upgrade` which is a DataStage Upgrade template pipeline.


# Usage Notes

## Jenkins Shared Libraries

All MettleCI pipeline examples make use of re-usable pipeline components, the terminology for which varies between technologies.

Jenkins is notable in that it is the only MettleCI-supported build system which requries its re-usable pipeline components to reside in a separate repository to the main repository (i.e. this one) which utilises those re-usable components.

For this reason you need to ensure that for Jenkins-based pipelines you do the following:

Deploy the jenkins-mci-shared-libraries to a separate Git repository, alongside the repository holding your DataStage assets (i.e. this repository)
Ensure that the first line of your Jenkins pipeline definition refers to the name of the repository you created to hold your shared libraries. e.g. @Library('jenkins-mci-shared-libraries') _


## Config Directory

When reviewing the supplied Jenkinsfile, you may notice that some steps refer to a "config" directory, 
which is not one of the directories present in the repository. This directory is created *on the fly* 
during the prior parameter substitution step:

```
bat label: 'Substitute parameters in DataStage config',
script: "${env.METTLE_SHELL} properties config 
 -baseDir datastage 
 -filePattern \"*.sh\"
 -filePattern \"DSParams\" 
 -filePattern \"Parameter Sets/*/*\" 
 -properties var.${params.environmentId} 
 -outDir config"
```

The `-outDir` parameter `config` generated the config directory which is then used in subsequent steps.

```
bat label: 'Transfer DataStage config and filesystem assets', 
script: "${env.METTLE_SHELL} remote upload 
 -host ${params.serverName} 
 -username ${datastageUsername} -password ${datastagePassword} 
 -transferPattern \"filesystem/**/*,config/*\" 
 -destination \"${env.BUILD_TAG}\""
```

## Referencing Multiple Repositories

The supplied Jenkinsfile assumes that the compliance rules are in the same repository as the project. Jenkins 
fetches the contents of the repository where the Jenkinsfile was sourced from *automatically* but other repos 
must be explicitly fetched. If you do not wish to place compliance rules in *every* project directory, you 
can place them in another repository and then clone that repo explicitly. Here is an example of that, taken 
from a different pipeline. (you may have to correct the indenting to use it in your pipeline)

```
stage('Static Analysis') {     
	steps {
		checkout([  
			$class: 'GitSCM', 
			branches: [[name: 'refs/heads/master']], 
			doGenerateSubmoduleConfigurations: false, 
			extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'compliance2']], 
			submoduleCfg: [], 
			userRemoteConfigs: [[ credentialsId: 'lar_at_demo_mettleci_i0', 
            	url:  'http://lar@demo.mettleci.io/bitbucket/scm/cr/compliance_ds115_icp35.git' ]]
				])
         bat label: 'Perform static analysis for non fatal violations', 
         	script: "${env.METTLE_SHELL} compliance test -assets datastage 
            	-report \"compliance_report_warn.xml\" 
                -junit -test-suite \"warnings\" 
                -ignore-test-failures  -rules compliance2\\WARN 
                -project-cache \"C:\\dm\\mci\\cache\\${params.serverName}\\${env.DATASTAGE_PROJECT}\""
        // bat label: 'Perform static analysis for fatal violations', 
        	script: "${env.METTLE_SHELL} compliance test -assets datastage 
            	-report \"compliance_report_fail.xml\" 
                -junit -test-suite \"failures\" 
                -rules compliance2\\FAIL 
                -project-cache \"C:\\dm\\mci\\cache\\${params.serverName}\\${env.DATASTAGE_PROJECT}\""
     }
     post {
         always {
             junit testResults: 'compliance_report_*.xml', allowEmptyResults: true
         }
     }
}  // end (sub) stage Static Analysis
```

Note also that this stage can run compliance twice, once for warnings and once for failures, (although failure is commented out) 
using seperate subfolders in the compliance repository. You would need to seperate your rules by which you want to fail the build 
and which you only want to warn users about. 
