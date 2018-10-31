def applicationName = 'my-php-app'
def baseImage  = "php";
def buildImage = "${baseImage}:latest";
def secretName = "git-repo-secret";

def buildImageNamespace = "openshift";
def buildImage = "${baseImage}:latest";
def secretName = "git-repo-secret";
def majorVersion = env.MAJOR_VERSION ?: "1";


pipeline {
    agent none
    options {
        timeout(time: 20, unit: 'MINUTES')
    }
    stages {
        stage('initialisation') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject() {
                            echo "Using project: ${openshift.project()}"
                        }
                    }
                }
            }
        }

        stage('create environments') {
            /*agent {
                label "maven"
            }*/
            steps {
                script {
                    openshift.withCluster() {
                      createProject(openshift, "${applicationName}-dev")
                      createProject(openshift, "${applicationName}-staging")
                      createProject(openshift, "${applicationName}-uat")
                      createProject(openshift, "${applicationName}-prod")
                    }
                }
            }
        }

        stage('sub-builds creation') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject() {
                            def bc = openshift.selector("bc", applicationName);
                            def workspace = pwd()
                            echo "Workspace ${workspace}"
                            if (!bc.exists()) {
                                echo "BC does not exist...creating it"
                                def scmUrl = scm.getUserRemoteConfigs()[0].getUrl();
                                openshift.newBuild("--strategy=source",
                                    "${buildImageNamespace}/${buildImage}~${scmUrl}",
                                    "--context-dir=${applicationName}", "--name=${applicationName}", "--source-secret=${secretName}",
                                    "--to=${applicationName}", "--env=GIT_SSL_NO_VERIFY=true")
                            } else {
                                echo "BC does exist...updating it if necessary"
                                def bcObject = bc.object();
                                def currentBuildImage = bcObject.spec.strategy.sourceStrategy.from.name;
                                def currentBuildImageNamespace = bcObject.spec.strategy.sourceStrategy.from.namespace;
                                if (currentBuildImage != buildImage || currentBuildImageNamespace != buildImageNamespace) {
                                   bcObject.spec.strategy.sourceStrategy.from.name = buildImage;
                                   bcObject.spec.strategy.sourceStrategy.from.namespace = buildImageNamespace;
                                   openshift.apply(bcObject);
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('build') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject() {
                            def build = openshift.selector("bc", applicationName);
                            build.startBuild();
                            def builds = build.related('builds')
                            timeout(5) {
                                builds.untilEach(1) {
                                    def status = it.object().status.phase;
                                    echo "Build in status: ${status}"
                                    return (status == "Complete" || status == "Cancelled")
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('tag') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject() {
                            openshift.tag("${applicationName}:latest", "${applicationName}:${majorVersion}.x")
                            openshift.tag("${applicationName}:latest", "${applicationName}-dev/${applicationName}:${majorVersion}.x")
                            openshift.tag("${applicationName}:latest", "${applicationName}-uat/${applicationName}:${majorVersion}.x")
                            openshift.tag("${applicationName}:latest", "${applicationName}-staging/${applicationName}:${majorVersion}.x")
                            openshift.tag("${applicationName}:latest", "${applicationName}-prod/${applicationName}:${majorVersion}.x")

                            def minorVersion = -1;
                            def istags = openshift.selector("is", applicationName).object().status.tags;
                            for( istag in istags ) {
                              echo "aaaaa ${istag}"
                              if( istag.tag.contains("${majorVersion}.")) {
                                minorVersion++;
                              }
                            }
                            openshift.tag("${applicationName}:latest", "${applicationName}:${majorVersion}.${minorVersion}")
                            openshift.tag("${applicationName}:latest", "${applicationName}-dev/${applicationName}:${majorVersion}.${minorVersion}")
                            //openshift.tag("${applicationName}:latest", "${applicationName}-staging:latest")
                            //openshift.tag("${applicationName}:latest", "${applicationName}:${baseImage}-${tag}")

                        }
                    }
                }
            }
        }
        stage('create application') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject("${applicationName}-dev") {
                        def dc = openshift.selector("dc", applicationName);
                            if (!dc.exists()) {
                                openshift.newApp(applicationName).narrow('svc').expose();
                            } else {
                                def route = openshift.selector("route", applicationName);
                                if (!route.exists()) {
                                  def svc = openshift.selector("svc", applicationName);
                                  svc.expose();
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('deploy') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject("${applicationName}-dev") {
                            def rm = openshift.selector("dc", applicationName).rollout()
                            timeout(5) {
                                openshift.selector("dc", applicationName).related('pods').untilEach(1) {
                                    return (it.object().status.phase == "Running")
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('template generation') {
            steps {
                script {
                    openshift.withCluster() {
                      createTemplate(openshift, applicationName, "dev");
                      //createTemplate(openshift, applicationName, "staging");
                      //createTemplate(openshift, applicationName, "uat");
                      //createTemplate(openshift, applicationName, "prod");
                    }
                }
            }
        }
        stage('commit template to SCM') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject("${applicationName}-dev") {
                          def templateObject = openshift.selector( "template", applicationName)
                          def checkoutDir =  pwd() + "@script/${applicationName}/";
                          def file = "${checkoutDir}/${applicationName}-template.yaml"
                          //echo "template ${templateObject}"
                          //echo "template ${templateObject}" > file
                          def now = new Date();
                          def date = now.format("yyyMMdd-HHmmss", TimeZone.getTimeZone('UTC'));
                          def workspace = pwd()
                          echo "Workspace ${workspace}"
                          //sh "echo ${templateObject} > ${file}; cd ${checkoutDir}; git add ${file}; git commit -m 'Adding template generated on $date'; git push"
                        }
                    }
                }
            }
        }
    }
}



def createProject(openshift, project) {
  try {
    openshift.newProject(project)
      } catch (Exception e) {
         if( !e.getMessage().contains("AlreadyExists")) throw e;
      }
}

def createTemplate(openshift, applicationName, environment ) {
  def project = "${applicationName}-${environment}";
  openshift.withProject(project) {
    def dc = openshift.selector("dc").objects(exportable:true)
    for( dcItem in dc ) {
      echo "DC Item After ${dcItem}"
      for( trigger in  dcItem.spec.triggers ) { 
        echo "Trigger ${trigger}"
        if( trigger.imageChangeParams != null ) {
	        trigger.imageChangeParams.from.namespace = "${applicationName}-\${ENVIRONMENT}";
    	    trigger.imageChangeParams.from.name = "${applicationName}:\${TAG}";

        }
      }
     }

    def svc = openshift.selector("svc").objects(exportable:true)
    //def cm = openshift.selector("cm").objects(exportable:true)
    def is = openshift.selector("is").objects(exportable:true)
    //def istag = openshift.selector("istag").objects(exportable:true)    
    def routes = openshift.selector("routes").objects(exportable:true)
    echo "Debug 1"
    for( route in routes ) {
      route.spec.host = null;
    }
    echo "Debug 2"    
    def templateObject = openshift.selector( "template", applicationName)
    if( templateObject.exists() ){
      templateObject.delete()
    }
    echo "Debug 3"    
    //def objects = dc + svc + cm + is + istag + routes;
    def objects = dc + svc + is + routes;
    echo "Debug 4"
    def params = [[ "name": "ENVIRONMENT" , "value": "{{ENVIRONMENT}}", "required": true, "description" : "Code environment (staging, uat, prod)"], 
                  [ "name": "TAG" ,         "value": "{{TAG}}", "required": true, "description" : "Tag to which the deployment points to"]]
    def template = [[ "kind":"Template", "apiVersion":"v1", "parameters": params, "objects": objects,
                       "metadata":[ "name":"${applicationName}", "annotations": [ "iconClass": "icon-php"],
                                     "labels":[ "template":"${applicationName}" ]]]]
    echo "Debug 5"
    openshift.create(template);
    echo "Debug 6"    
  }
}








