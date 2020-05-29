import groovy.json.JsonOutput

config_url = "https://github.com/conan-ci-cd-training/settings.git"

artifactory_metadata_repo = "conan-metadata"
conan_develop_repo = "conan-develop"
conan_tmp_repo = "conan-tmp"
artifactory_url = (env.ARTIFACTORY_URL != null) ? "${env.ARTIFACTORY_URL}" : "jfrog.local"

build_order_file = "bo.json"
build_order = null

profiles = [
  "debug-gcc6": "conanio/gcc6",	
  "release-gcc6": "conanio/gcc6"	
]

build_result = [:]

user = "mycompany"
channel = "stable"

products = ["App/1.0@mycompany/stable", "App2/1.0@mycompany/stable"]

affected_products = []

def get_stages(product, profile, docker_image) {
  return {
    stage(profile) {
      node {
        docker.image(docker_image).inside("--net=host") {
          echo "Inside docker image: ${docker_image}"
          withEnv(["CONAN_USER_HOME=${env.WORKSPACE}/conan_home"]) {
            try {
              stage("Configure conan") {
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
                // In the case this job was triggered after a merge to the library's develop branch
                // we upload to tmp, and later if everything is OK will promote to develop
                stage("Upload to conan-tmp") {
                  if(params.library_branch == "develop") {
                    sh "conan upload '*' --all -r ${conan_tmp_repo} --confirm"
                  }
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

def promote_with_lockfile(lockfile_json, source_repo, target_repo, additional_references=[]) {
  // additional_references are for the case, like in this workflow
  // that we have a lockfile with some nodes marked as build but they
  // are the result of creating a new revision of another lib in the pipeline
  // that triggered this one. That library is not going to be in the lockfile
  // marked as build so maybe we want to add some references to the promotion
  def references_to_copy = []
  def nodes = lockfile_json['graph_lock'].nodes
  nodes.each { id, node_info ->
    if (node_info.modified) {
      references_to_copy.add(node_info.pref)
    }
    else {
      additional_references.each { reference ->
        if (node_info.pref.startsWith(reference)) {
          references_to_copy.add(node_info.pref)
        }
      }
    }
  }
  // now move all the package references that were build
  references_to_copy.each { pref ->
    // path: repo/user/name/version/channel/rrev/package/pkgid/prev/conan_package.tgz
    echo "copy ${pref} from ${source_repo} to ${target_repo}"
    name_version = pref.split("#")[0].split("@")[0]
    rrev = pref.split("#")[1].split(":")[0]
    def pkgid = pref.split("#")[1].split(":")[1]
    def prev = pref.split("#")[2]
    echo "copy name: ${name_version} to ${target_repo}"
    echo "copy rrev: ${rrev} to ${target_repo}"
    echo "copy pkgid: ${pkgid} to ${target_repo}"
    echo "copy prev: ${prev} to ${target_repo}"
    withCredentials([usernamePassword(credentialsId: 'artifactory-credentials', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
      sh "curl -u\"\${ARTIFACTORY_USER}\":\"\${ARTIFACTORY_PASSWORD}\" -XPOST \"http://${artifactory_url}:8081/artifactory/api/copy/${source_repo}/${user}/${name_version}/${channel}/${rrev}/export?to=${target_repo}/${user}/${name_version}/${channel}/${rrev}\""
    } 
    withCredentials([usernamePassword(credentialsId: 'artifactory-credentials', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
      sh "curl -u\"\${ARTIFACTORY_USER}\":\"\${ARTIFACTORY_PASSWORD}\" -XPOST \"http://${artifactory_url}:8081/artifactory/api/copy/${source_repo}/${user}/${name_version}/${channel}/${rrev}/package/${pkgid}/${prev}?to=${target_repo}/${user}/${name_version}/${channel}/${rrev}/package/${pkgid}/${prev}\""
    } 
  }
}

pipeline {
  agent none

  parameters {
    string(name: 'reference',)
    string(name: 'library_branch',)
  }

  stages {
    stage('Build affected products') {
      agent any
      steps {
        script {          
          products_build_result = products.collectEntries { product ->
            stage("Build ${product}") {
              echo "Building product '${product}'"
              echo " - for changes in '${params.reference}'"
              build_result = parallel profiles.collectEntries { profile, docker_image ->
                ["${profile}": get_stages(product, profile, docker_image)]
              }              
            }
            ["${product}": build_result]
          }
          println products_build_result
        }
      }
    }

    stage("Upload to develop repo") {
      when {expression { return params.library_branch == "develop" }}
      steps {
        script {
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

                // promote using each of the lockfiles
                products_build_result.each { product, result ->
                  result.each { profile, lockfile ->
                    if (lockfile.size()>0) {
                      stage("Upload artifacts built with profile: ${profile} to conan-develop repo") {
                        promote_with_lockfile(lockfile, conan_tmp_repo, conan_develop_repo, ["${params.reference}"])
                      }
                      stage("Upload ${product} lockfile for profile: ${profile}") {
                        writeJSON file: "conan.lock", json: lockfile
                        def lockfile_url = "http://${artifactory_url}:8081/artifactory/${artifactory_metadata_repo}/${env.JOB_NAME}/${env.BUILD_NUMBER}/${product}/${profile}/conan.lock"
                        echo "${lockfile_url}"
                        sh "printenv"
                        withCredentials([usernamePassword(credentialsId: 'artifactory-credentials', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
                          sh "curl --user \"\${ARTIFACTORY_USER}\":\"\${ARTIFACTORY_PASSWORD}\" -X PUT ${lockfile_url} -T conan.lock"
                        }
                      }                            
                    }
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
