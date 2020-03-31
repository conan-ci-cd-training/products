import groovy.json.JsonOutput

def config_url = "https://github.com/conan-ci-cd-training/settings.git"

def artifactory_metadata_repo = "conan-develop-metadata"
def conan_develop_repo = "conan-develop"
def conan_tmp_repo = "conan-tmp"
def artifactory_url = (env.ARTIFACTORY_URL != null) ? "${env.ARTIFACTORY_URL}" : "jfrog.local"

def build_order_file = "bo.json"
def build_order = null

def profiles = [
  "conanio-gcc8": "conanio/gcc8",	
  "conanio-gcc7": "conanio/gcc7"	
]

def build_result = [:]

def products = [
  "App": "https://github.com/conan-ci-cd-training/App.git",	
  "App2": "https://github.com/conan-ci-cd-training/App2.git"	
]

def get_stages(profile, docker_image, config_url, conan_develop_repo, conan_tmp_repo, library_branch, artifactory_url) {
  return {
    stage(profile) {
      node {
        docker.image(docker_image).inside("--net=host") {
          echo "Inside the docker"
          withEnv(["CONAN_USER_HOME=${env.WORKSPACE}/conan_home"]) {
            try {
              stage("Configure conan") {
                sh "printenv"
                sh 'rm -rf "${CONAN_USER_HOME}"'
                sh "conan --version"
                sh "conan config install ${config_url}"
                sh "conan remote add ${conan_develop_repo} http://${artifactory_url}:8081/artifactory/api/conan/${conan_develop_repo}" // the namme of the repo is the same that the arttifactory key
                sh "conan remote add ${conan_tmp_repo} http://${artifactory_url}:8081/artifactory/api/conan/${conan_tmp_repo}" // the namme of the repo is the same that the arttifactory key
                withCredentials([usernamePassword(credentialsId: 'artifactory', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
                  sh "conan user -p ${ARTIFACTORY_PASSWORD} -r ${conan_develop_repo} ${ARTIFACTORY_USER}"
                  sh "conan user -p ${ARTIFACTORY_PASSWORD} -r ${conan_tmp_repo} ${ARTIFACTORY_USER}"
                }
              }
              // create the graph lock for the latest versions of dependencies in develop repo and with the
              // latest version if the library that trigered this pipeline
              // as graph lock iterates through all the remotes to get the last version
              // the last version of the dependant library will be retreived from conan-build
              // what happens if there's another version uploaded in conan-build that is not
              // the one that triggered this pipeline? then, to solve this:
              // 1 - create the graph lock taking dependencies from conan-develop -> we create a lockfile
              // 2 - install the dependencies of that lockfile to the local cache
              // 3 - install the exact reference we have just passed to the pipeline from remote=conan-build
              // 4 - now we have in the cache all the exact references we wanted, we create a graph lock with the contents of the cache
              // the final lockfile we are searching for: the one that has all the latest from develop
              // plus the one we have just created
              def lockfile = "conan.lock"
              def buildInfoFilename = "${profile}.json"
              stage("Create graph lock for ${params.project_id}") {
                sh "conan graph lock ${params.project_id}  --profile ${profile} --lockfile=${lockfile} -r ${conan_develop_repo}"
                sh "cat ${lockfile}"
              }
              stage("Insert the new revision ${params.reference} in ${params.project_id} graph") {
                sh "conan install ${params.project_id} --lockfile=${lockfile} --profile ${profile} -r ${conan_develop_repo}"
                sh "conan install ${params.reference} --update --profile ${profile} -r ${conan_tmp_repo}"
                sh "conan search --revisions"
                //sh "conan remote disable '*'"
                sh "conan graph lock ${params.project_id} --profile ${profile} --lockfile=${lockfile}"
                sh "cat ${lockfile}"
                sh "conan install ${params.project_id}  --profile ${profile} --build missing --lockfile=${lockfile}"
              }
              stage("Start build info: ${params.build_name} ${params.build_number}") { 
                sh "conan_build_info --v2 start ${params.build_name} ${params.build_number}"
              }
              stage("Upload") {
                sh "conan upload '*' --all -r ${conan_tmp_repo} --confirm  --force"
                if (library_branch == "master") {
                  sh "conan upload '*' --all -r ${conan_develop_repo} --confirm  --force"
                }
              }
              stage("Create build info") {
                withCredentials([usernamePassword(credentialsId: 'artifactory', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
                  sh "conan_build_info --v2 create --lockfile ${lockfile} --user \"\${ARTIFACTORY_USER}\" --password \"\${ARTIFACTORY_PASSWORD}\" ${buildInfoFilename}"
                  buildInfo = readJSON(file: buildInfoFilename)
                  sh "cat ${buildInfoFilename}"
                }
              }                            
              return buildInfo              
            }
            finally {
                deleteDir()
            }
          }
        }
      }
    }
  }
}

pipeline {
  agent none

  parameters {
    string(name: 'reference',)
    string(name: 'organization',)
    string(name: 'build_name',)
    string(name: 'build_number',)
    string(name: 'commit_number',)
    string(name: 'library_branch',)
  }

  stages {

    stage("Check affected products") {
      steps {
        script {
          String docker_image = profiles.values().iterator().next()
          String profile = profiles.keySet().iterator().next()
          docker.image(docker_image).inside("--net=host") {
            sh "conan config install ${config_url}"
            sh "conan remote add ${conan_develop_repo} http://${artifactory_url}:8081/artifactory/api/conan/${conan_develop_repo}" // the namme of the repo is the same that the arttifactory key
            sh "conan remote add ${conan_tmp_repo} http://${artifactory_url}:8081/artifactory/api/conan/${conan_tmp_repo}" // the namme of the repo is the same that the arttifactory key
            withCredentials([usernamePassword(credentialsId: 'artifactory', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
              sh "conan user -p ${ARTIFACTORY_PASSWORD} -r ${conan_develop_repo} ${ARTIFACTORY_USER}"
              sh "conan user -p ${ARTIFACTORY_PASSWORD} -r ${conan_tmp_repo} ${ARTIFACTORY_USER}"
            }
            products.each { product ->
              println "name: ${product.key} repo: ${product.value}"
              def lockfile = "${product.lock}"
              def bo = "${product.json}"
              sh "conan install ${product.key} --profile ${profile} -r ${conan_develop_repo}"
              sh "conan graph lock ${product.key} --profile ${profile} --lockfile=${lockfile} -r ${conan_develop_repo}"
              sh "conan graph build-order ${lockfile} --json=${bo}"
              sh "cat ${bo}"
            }
          }
        }
      }
    }

    stage('Build information') {
      steps {
        script {
          echo "Building project '${params.organization}/${params.project_id}'"
          echo " - for changes in '${params.reference}'"
          echo " - called from: ${params.build_name} ${params.build_number}"
          echo " - commit: ${params.commit_number} from branch: ${params.library_branch}"
        }
      }
    }

    stage('Build') {
      agent any
      steps {
        script {
          build_result = withEnv(["CONAN_HOOK_ERROR_LEVEL=40"]) {
            parallel profiles.collectEntries { profile, docker_image ->
              ["${profile}": get_stages(profile, docker_image, config_url, conan_develop_repo, conan_tmp_repo, params.library_branch, artifactory_url)]
            }
          }         
        }
      }
    }

    stage("Promote") {
      steps {
        script {
          docker.image("conanio/gcc8").inside("--net=host") {
            def last_info = ""
            build_result.each { profile, buildInfo ->
              writeJSON file: "${profile}.json", json: buildInfo
              if (last_info != "") {
                sh "conan_build_info --v2 update ${profile}.json ${last_info} --output-file mergedbuildinfo.json"
              }
              last_info = "${profile}.json"
            }                    
            sh "cat mergedbuildinfo.json"
            withCredentials([usernamePassword(credentialsId: 'artifactory', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
              sh "conan_build_info --v2 publish --url http://${artifactory_url}:8081/artifactory --user \"\${ARTIFACTORY_USER}\" --password \"\${ARTIFACTORY_PASSWORD}\" mergedbuildinfo.json"
            }
            // instead of:
            // sh "conan upload '*' --all -r ${conan_tmp_repo} --confirm  --force"
            // if (library_branch == "master") {       
            //   sh "curl -fL https://getcli.jfrog.io | sh"
            //   withCredentials([usernamePassword(credentialsId: 'artifactory', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
            //     sh "./jfrog rt c artifactory --url=http://${artifactory_url}:8081/artifactory --user=\"\${ARTIFACTORY_USER}\" --password=\"\${ARTIFACTORY_PASSWORD}\""
            //   }
            //   sh "./jfrog rt bpr \"${params.build_name}\" \"${params.build_number}\" conan-develop --include-dependencies"
            // }        
          }
        }
      }
    }      
  }
}
