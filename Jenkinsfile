def applicationName = 'php-simple-app'
def baseImage  = "php";
def buildImage = "${baseImage}:latest";
def secretName = "git-repo-secret";

def createProject(openshift, project) {
  try {
    openshift.newProject(project)
      } catch (Exception e) {
         if( !e.getMessage().contains("AlreadyExists")) throw e;
      }
}



pipeline {
    agent none
    options {
        timeout(time: 20, unit: 'MINUTES')
    }
    stages {
        stage('preamble') {
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
                            if (!bc.exists()) {
                                echo "BC does not exist...creating it"
                                def scmUrl = scm.getUserRemoteConfigs()[0].getUrl();
                                openshift.newBuild("--strategy=source",
                                    "${buildImage}~${scmUrl}",
                                    "--context-dir=${applicationName}", "--name=${applicationName}", "--source-secret=${secretName}",
                                    "--to=${applicationName}", "--env=GIT_SSL_NO_VERIFY=true")
                            } else {
                                echo "BC does exist...updating it if necessary"
                                def bcObject = bc.object();
                                def currentBuildImage = bcObject.spec.strategy.sourceStrategy.from.name;
                                if (currentBuildImage != buildImage ) {
                                   bcObject.spec.strategy.sourceStrategy.from.name = buildImage;
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
                           /* def build = openshift.selector("bc", applicationName);
                            build.startBuild();
                            def builds = build.related('builds')
                            timeout(5) {
                                builds.untilEach(1) {
                                    return (it.object().status.phase == "Complete")
                                }
                            }
                        */}
                    }
                }
            }
        }
        stage('tag') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject() {
                            def now = new Date();
                            def tag = now.format("yyyMMdd-HHmmss", TimeZone.getTimeZone('UTC'));
                            openshift.tag("${applicationName}:latest", "${applicationName}-dev/${applicationName}:latest")
                            openshift.tag("${applicationName}:latest", "${applicationName}-staging/${applicationName}:latest")
                            openshift.tag("${applicationName}:latest", "${applicationName}-uat/${applicationName}:latest")
                            openshift.tag("${applicationName}:latest", "${applicationName}-prod/${applicationName}:latest")
                            openshift.tag("${applicationName}:latest", "${applicationName}-staging:latest")
                            openshift.tag("${applicationName}:latest", "${applicationName}:${baseImage}-${tag}")

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
                        openshift.withProject() {
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
                        def dc, svc, cm, is, routes;
                        openshift.withProject() {
                          dc = openshift.selector("dc").objects(exportable:true)
                          svc = openshift.selector("svc").objects(exportable:true)
                          cm = openshift.selector("cm").objects(exportable:true)
                          is = openshift.selector("is").objects(exportable:true)
                          routes = openshift.selector("routes").objects(exportable:true)
                        }
                        openshift.withProject("${applicationName}-dev") {
                          def templateObject = openshift.selector( "template", applicationName)
                          if( templateObject.exists() ){
                            templateObject.delete()
                          }
                          def objects = dc + svc + cm + is + routes;
                          def template = [[ "kind":"Template", "apiVersion":"v1", "objects": objects,
                                             "metadata":[ "name":"${applicationName}", "annotations": [ "iconClass": "icon-php"],
                                                           "labels":[ "template":"${applicationName}" ]]]]
                          openshift.create(template);
                        }
                    }
                }
            }
        }
    }
}
