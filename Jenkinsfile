// define project name
def dev_project_name = 'ubip-dev'
def qa_project_name = 'ubip-qa'

// define dc patch
def dev_dc_patch_file = 'src/main/fabric8/deployment-dev.yml'
def dev_dc_patch
def qa_dc_patch_file = 'src/main/fabric8/deployment-qa.yml'
def qa_dc_patch

// define git info
def gitlab_credentialsId = 'nprod-deployer'
def sonartokenId = 'SONAR_ADMIN_TOKEN'
def gitlab_url = "http://gitlab-devops-tools.apps.paas-test.ubrmb.com/${GIT_GROUP_NAME}/${app_name}.git"
def sonar_url = 'http://sonarqube.devops-tools.svc.cluster.local:9000'

def version
def service_name = "${app_name}"

pipeline {
    agent {
        label 'maven'
    }
    options {
        timeout(time:1, unit: 'HOURS')
    }

    environment {
        soanr_admin_token = credentials("${sonartokenId}")
    }

    stages {
        stage('Build Package') {
            steps {
                git branch: "${env.GIT_BRANCH}", credentialsId: "${gitlab_credentialsId}", url: "${gitlab_url}"
                script {
                    def pom = readMavenPom file: 'pom.xml'
                    version = pom.version
  		            dev_dc_patch = readFile "${dev_dc_patch_file}"
  		            qa_dc_patch = readFile "${qa_dc_patch_file}"
                }
                configFileProvider([configFile(fileId: 'b66e48e5-8d8e-4c32-b88d-cdc0a6e45339', variable: 'MAVEN_SETTINGS')]) {
                    sh "mvn -s $MAVEN_SETTINGS clean install -DskipTests=true"
			    }
            }
        }
      
        stage('Junit Test') {
            steps {
                configFileProvider([configFile(fileId: 'b66e48e5-8d8e-4c32-b88d-cdc0a6e45339', variable: 'MAVEN_SETTINGS')]) {
                    sh "mvn -s $MAVEN_SETTINGS test ${MVN_JAVA_OPTION}"
				}
                step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
            }
        }
      
        stage('Code Analysis') {
            steps {
			    configFileProvider([configFile(fileId: 'b66e48e5-8d8e-4c32-b88d-cdc0a6e45339', variable: 'MAVEN_SETTINGS')]) {
                    sh "mvn -s $MAVEN_SETTINGS sonar:sonar -Dsonar.login=${soanr_admin_token} -Dsonar.host.url=${sonar_url} -DskipTests=true -Dsonar.java.coveragePlugin=jacoco -Dsonar.jacoco.reportPath=target/jacoco.exec -Dsonar.dynamicAnalysis=reuseReports -Dsonar.jacoco.reportMissing.force.zero=true"
				}
				sleep 10
            }
        }
    
        stage('Check Analysis Result') {
            steps {
				script {
					if ("${CHECK_ANALYSIS_RESULT_FLG}" == 'true') {
						sh """
							curl -u ${SONAR_TOKEN}: http://sonarqube-devops-tools.apps.paas-test.ubrmb.com/api/qualitygates/project_status?projectKey="$SONAR_PROJECT_KEY" | cut -b 29-30 >> ./rs.tmp
							cat ./rs.tmp
							if [ \$(<./rs.tmp) == "OK" ]
							then
								echo "Code Analysis Result is OK"
							else
								echo "Code Analysis Result is ERROR"
								exit 1
							fi
						"""
					} else {
						echo 'Skip Check Analysis Result stage'
					}
				}
            }
        }
	
        stage('Archive App') {
            steps {
			    configFileProvider([configFile(fileId: 'b66e48e5-8d8e-4c32-b88d-cdc0a6e45339', variable: 'MAVEN_SETTINGS')]) {
                    sh "mvn -s $MAVEN_SETTINGS deploy -DskipTests=true -P nexus3"
				}
            }
        }
      
        stage('Create Image Builder') {
            when {
                expression {
                    openshift.withCluster() {
                        openshift.withProject("${dev_project_name}") {
                            return !openshift.selector("bc", "${service_name}").exists();
                        }
                    }
                }
            }
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject("${dev_project_name}") {
                            openshift.newBuild("--name=${service_name}", "--image-stream=openshift/fuse7-java-openshift:1.2", "--binary=true")
                        }
                    }
                }
            }
        }
      
        stage('Build Image') {
            steps {
                sh "rm -rf oc-build && mkdir -p oc-build/deployments"
                sh "cp target/*.jar oc-build/deployments/"
                script {
                    openshift.withCluster() {
                        openshift.withProject("${dev_project_name}") {
                            openshift.selector("bc", "${service_name}").startBuild("--from-dir=oc-build", "--wait=true")
                        }
                    }
                }
            }
        }
      
        stage('Deploy DEV') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject("${dev_project_name}") {
                            if (!openshift.selector('dc', "${service_name}").exists()) {
                                def app = openshift.newApp("${service_name}:latest")
                                def dc = openshift.selector("dc", "${service_name}")
                                echo "dev_dc_patch: ${dev_dc_patch}"
                                dc.patch("\"${dev_dc_patch}\"")
                                while (dc.object().spec.replicas != dc.object().status.availableReplicas) {
                                    sleep 10
                                }
                                openshift.set("triggers", "dc/${service_name}", "--manual") 
                                app.narrow("svc").delete();
                                dc.expose("--port=8080");
                                app.narrow("svc").expose("--port=8080")
                            } else {
								def dc = openshift.selector("dc", "${service_name}")
                                echo "dev_dc_patch: ${dev_dc_patch}"
								dc.patch("\"${dev_dc_patch}\"")
                                openshift.selector("dc", "${service_name}").rollout().latest();
                            } 
                        }
                    }
                }
            }
        }

        stage('Tag QA image') {
            steps {
				script {
					if ("${DEPLOY_QA_FLG}" == 'true') {
						openshift.withCluster() {
							openshift.tag("${dev_project_name}/${service_name}:latest", "${qa_project_name}/${service_name}:${version}")
						}
						sleep 5
					} else {
						echo 'Skip Tag QA image stage'
					}
				}
            }
        }
     
        stage('Deploy QA') {
            steps {
                script {
					if ("${DEPLOY_QA_FLG}" == 'true') {
						openshift.withCluster() {
							openshift.withProject("${qa_project_name}") {
								dcselector = openshift.selector('dc', "${service_name}")
								if (!dcselector.exists()) {          
									def app = openshift.newApp("${service_name}:${version}")
									def dc = openshift.selector("dc", "${service_name}")
									echo "qa_dc_patch: ${qa_dc_patch}"
									dc.patch("\"${qa_dc_patch}\"")
									while (dc.object().spec.replicas != dc.object().status.availableReplicas) {
										sleep 10
									}
									openshift.set("triggers", "dc/${service_name}", "--manual") 
									app.narrow("svc").delete();
									dc.expose("--port=8080");
									app.narrow("svc").expose("--port=8080")
								} else {
									def latestDeploymentVersion = dcselector.object().status.latestVersion
									def rcobject = openshift.selector('rc', "${service_name}-${latestDeploymentVersion}").object()
									while (dcselector.object().spec.replicas != dcselector.object().status.availableReplicas) {
										sleep 10
									}
									openshift.set("triggers", "dc/${service_name}", "--remove-all")
									openshift.set("triggers", "dc/${service_name}", "--from-config")
									openshift.set("triggers", "dc/${service_name}", "--containers=${service_name}", "--from-image=${qa_project_name}/${service_name}:${version}", "--auto")
								}
							
							}
						}
					} else {
						echo 'Skip Deploy QA stage'
					}
                }
            }
        }

    }
}
