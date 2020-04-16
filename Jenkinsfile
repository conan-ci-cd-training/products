import groovy.json.JsonOutput

def config_url = "https://github.com/conan-ci-cd-training/settings.git"

def artifactory_metadata_repo = "conan-develop-metadata"
def conan_develop_repo = "conan-develop"
def conan_tmp_repo = "conan-tmp"
def artifactory_url = (env.ARTIFACTORY_URL != null) ? "${env.ARTIFACTORY_URL}" : "jfrog.local"

def build_order_file = "bo.json"
def build_order = null

def profiles = [
  "debug-gcc6": "conanio/gcc6",	
  "release-gcc6": "conanio/gcc6"	
]

def build_result = [:]

def user = "mycompany"
def channel = "stable"

def products = [
  "App/1.0@mycompany/stable": "https://github.com/conan-ci-cd-training/App.git",	
  "App2/1.0@mycompany/stable": "https://github.com/conan-ci-cd-training/App2.git"	
]

def affected_products = []

def get_stages(product, profile, docker_image, config_url, conan_develop_repo, conan_tmp_repo, library_branch, artifactory_url) {
  return {
    stage(profile) {
      node {
        docker.image(docker_image).inside("--net=host") {
          echo "Inside the docker"
          withEnv(["CONAN_USER_HOME=${env.WORKSPACE}/conan_home"]) {
            try {
              stage("Configure conan") {
                sh "printenv"
                //sh 'rm -rf "${CONAN_USER_HOME}"'
                sh "conan --version"
                sh "conan config install ${config_url}"
                sh "conan remote add ${conan_develop_repo} http://${artifactory_url}:8081/artifactory/api/conan/${conan_develop_repo}" // the namme of the repo is the same that the arttifactory key
                sh "conan remote add ${conan_tmp_repo} http://${artifactory_url}:8081/artifactory/api/conan/${conan_tmp_repo}" // the namme of the repo is the same that the arttifactory key
                withCredentials([usernamePassword(credentialsId: 'artifactory-credentials', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
                  sh "conan user -p ${ARTIFACTORY_PASSWORD} -r ${conan_develop_repo} ${ARTIFACTORY_USER}"
                  sh "conan user -p ${ARTIFACTORY_PASSWORD} -r ${conan_tmp_repo} ${ARTIFACTORY_USER}"
                }
              }
              // create the graph lock for the latest versions of dependencies in develop repo and with the
              // latest revision of the library that trigered this pipeline
              def lockfile = "conan.lock"
              def bo_file = "build_order.json"
              def affected_product = false
              def lock_contents = [:]
              stage("Check if the new revision ${params.reference} is in ${product} graph") {
                // this is a workaround, because installing with a specific reference does not create a lockfile
                // and also, we need the information of the build nodes
                // develop should be consistent without missing packages
                
                // install just the recipe for the new revision of the lib and then resolve graph with develop repo
                // we don't want to bring binaries at this moment, this could be just a agent that
                // then sends lockfiles to other nodes to build them
                sh "conan download ${params.reference} -r ${conan_tmp_repo} --recipe"
                sh "conan graph lock ${product} --profile=${profile} --lockfile=${lockfile} -r ${conan_develop_repo}"

                // check if the build-order is empty, this product may not be affected by the changes
                // or maybe we don't have to build anything if we are relaunching the builds
                sh "conan graph build-order ${lockfile} --json=${bo_file} --build missing"
                build_order = readJSON(file: bo_file)
                if (build_order.size()>0) {
                  affected_product = true
                }
              }
              if (affected_product) {
                stage("Build ${product}")
                {
                  // now that we have a lockfile as an input conan install will update the build nodes
                  sh "conan install ${product} --profile ${profile} --lockfile=${lockfile} --build missing "
                  lock_contents = readJSON(file: lockfile)
                  sh "cat ${lockfile}"
                }
                stage("Upload to conan-tmp") {
                  // we upload to tmp, and later if everything is OK will promote to develop
                  sh "conan upload '*' --all -r ${conan_tmp_repo} --confirm  --force"
                }
              }
              return lock_contents              
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
    stage('Build affected products') {
      agent any
      steps {
        script {          
          //products_build_result = affected_products.collectEntries { product ->
          products_build_result = products.collectEntries { product, repo ->
            stage("Build ${product}") {
              echo "Building product '${product}'"
              echo " - for changes in '${params.reference}'"
              echo " - called from: ${params.build_name}, build number: ${params.build_number}"
              echo " - commit: ${params.commit_number} from branch: ${params.library_branch}"              
              build_result = parallel profiles.collectEntries { profile, docker_image ->
                ["${profile}": get_stages(product, profile, docker_image, config_url, conan_develop_repo, conan_tmp_repo, params.library_branch, artifactory_url)]
              }              
            }
            ["${product}": build_result]
          }
          println products_build_result
        }
      }
    }

    stage("Upload to develop repo") {
      steps {
        script {
          if (library_branch == "develop") {  
            docker.image("conanio/gcc6").inside("--net=host") {
              // promote libraries to develop
              withEnv(["CONAN_USER_HOME=${env.WORKSPACE}/conan_cache"]) {
                try {
                  sh "conan config install ${config_url}"
                  sh "conan remote add ${conan_develop_repo} http://${artifactory_url}:8081/artifactory/api/conan/${conan_develop_repo}" // the namme of the repo is the same that the arttifactory key
                  sh "conan remote add ${conan_tmp_repo} http://${artifactory_url}:8081/artifactory/api/conan/${conan_tmp_repo}" // the namme of the repo is the same that the arttifactory key
                  withCredentials([usernamePassword(credentialsId: 'artifactory-credentials', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
                      sh "conan user -p ${ARTIFACTORY_PASSWORD} -r ${conan_develop_repo} ${ARTIFACTORY_USER}"
                      sh "conan user -p ${ARTIFACTORY_PASSWORD} -r ${conan_tmp_repo} ${ARTIFACTORY_USER}"
                  }
                  // some kind of manual promotion
                  // take all the lockfiles and copy all the revisions marked as built to conan-develop
                  // ALSO copy the revision we injected here from the caller pipeline
                  def references_to_copy = []
                  products_build_result.each { product, result ->
                    result.each { profile, lockfile ->
                    if (lockfile.size()>0) {
                        def nodes = lockfile['graph_lock'].nodes
                        nodes.each { id, node_info ->
                          if (node_info.modified) {
                            references_to_copy.add(node_info.pref)
                          }
                        }
                        stage("Upload ${product} lockfile for ${profile}") {
                          writeJSON file: "conan.lock", json: lockfile
                          def lockfile_url = "http://${artifactory_url}:8081/artifactory/${artifactory_metadata_repo}/${product}/${profile}/conan.lock"
                          def lockfile_sha1 = sha1(file: "conan.lock")
                          withCredentials([usernamePassword(credentialsId: 'artifactory-credentials', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
                            sh "curl --user \"\${ARTIFACTORY_USER}\":\"\${ARTIFACTORY_PASSWORD}\" --header 'X-Checksum-Sha1:'${lockfile_sha1} --header 'Content-Type: application/json' ${lockfile_url} --upload-file conan.lock"
                          }                                
                          lock_contents = readJSON(file: lockfile)
                        }                            
                      }
                    }
                  }
                  references_to_copy.unique()

                  // first move the new revision of libB we have just created
                  // TODO: better to take from the lockfiles the package revisions of libB with the new revision
                  // to make sure we don't put package revisions we don't want in the develop repo
                  echo "copy ${params.reference} to conan-develop"
                  def name = params.reference.split("#")[0].split("@")[0]
                  def rrev = params.reference.split("#")[1]
                  withCredentials([usernamePassword(credentialsId: 'artifactory-credentials', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
                    sh "curl -u\"\${ARTIFACTORY_USER}\":\"\${ARTIFACTORY_PASSWORD}\" -XPOST \"http://${artifactory_url}:8081/artifactory/api/copy/${conan_tmp_repo}/${user}/${name}/${channel}/${rrev}?to=${conan_develop_repo}/${user}/${name}/${channel}\""
                  }                   

                  // now move all the package references that were build
                  references_to_copy.each { pref ->
                    // path: repo/user/name/version/channel/rrev/package/pkgid/prev/conan_package.tgz
                    echo "copy ${pref} to conan-develop"
                    name_version = pref.split("#")[0].split("@")[0]
                    rrev = pref.split("#")[1].split(":")[0]
                    def pkgid = pref.split("#")[1].split(":")[1]
                    def prev = pref.split("#")[2]
                    echo "copy name: ${name_version} to conan-develop"
                    echo "copy rrev: ${rrev} to conan-develop"
                    echo "copy pkgid: ${pkgid} to conan-develop"
                    echo "copy prev: ${prev} to conan-develop"
                    withCredentials([usernamePassword(credentialsId: 'artifactory-credentials', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
                      sh "curl -u\"\${ARTIFACTORY_USER}\":\"\${ARTIFACTORY_PASSWORD}\" -XPOST \"http://${artifactory_url}:8081/artifactory/api/copy/${conan_tmp_repo}/${user}/${name_version}/${channel}/${rrev}/export?to=${conan_develop_repo}/${user}/${name_version}/${channel}/${rrev}\""
                    } 
                    withCredentials([usernamePassword(credentialsId: 'artifactory-credentials', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
                      sh "curl -u\"\${ARTIFACTORY_USER}\":\"\${ARTIFACTORY_PASSWORD}\" -XPOST \"http://${artifactory_url}:8081/artifactory/api/copy/${conan_tmp_repo}/${user}/${name_version}/${channel}/${rrev}/package/${pkgid}/${prev}?to=${conan_develop_repo}/${user}/${name_version}/${channel}/${rrev}/package/${pkgid}/${prev}\""
                    } 
                  }
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
  }
}
