def applicationName = 'php-simple-app'
def baseImage  = "php";
def buildImage = "${baseImage}:latest";
def secretName = "git-repo-secret";
 
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
                            def build = openshift.selector("bc", applicationName);
                            build.startBuild();
                            def builds = build.related('builds')
                            timeout(5) {
                                builds.untilEach(1) {
                                    return (it.object().status.phase == "Complete")
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('create') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject() {
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
        stage('tag') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject() {
                            def now = new Date();
                            def tag = now.format("yyyMMdd-HHmmss", TimeZone.getTimeZone('UTC'));
                            openshift.tag("${applicationName}:latest", "${applicationName}-staging:latest")
                            openshift.tag("${applicationName}:latest", "${applicationName}:${baseImage}-${tag}")
 
                        }
                    }
                }
            }
        }
        stage('generate and commit template') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject() {
                          def maps = openshift.selector('dc')
                          def objects = maps.objects( exportable:true )
                          echo "Export des objets: ${objects}"
                        }
                    }
                }
            }
        }
    }
}
 
