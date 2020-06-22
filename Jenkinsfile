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

def build_ref_with_lockfile(reference, lockfile, profile, upload_ref) {
  return {
    stage("Build: ${reference} - ${profile}") {
      unstash lockfile
      def actual_reference_name = reference.split("/")[0]
      def recipe_reference_with_revision = reference.split(":")[0]
      def actual_reference = reference.split("#")[0]
      echo("Build ${actual_reference_name}")
      sh "cp ${lockfile} conan.lock"
      sh "conan install ${actual_reference} --build ${actual_reference_name} --lockfile conan.lock"
      sh "mv conan.lock ${actual_reference_name}-${profile}.lock"
      stash name: "${actual_reference_name}-${profile}.lock", includes: "${actual_reference_name}-${profile}.lock"
      stage ("Upload reference ${actual_reference}-${profile} to ${conan_tmp_repo}") {
        if (upload_ref) {
          sh "conan upload ${actual_reference} --all -r ${conan_tmp_repo} --confirm"
        }
      }
    }
  }
}

def get_stages(product, profile, docker_image) {
  return {
    stage(profile) {
      node {
        docker.image(docker_image).inside("--net=host") {
          echo "Inside docker image: ${docker_image}"
          withEnv(["CONAN_USER_HOME=${env.WORKSPACE}/${profile}/conan_home"]) {
            try {
              stage("Configure conan") {
                sh "conan config install ${config_url}"
                withCredentials([usernamePassword(credentialsId: 'artifactory-credentials', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
                    sh "conan user -p ${ARTIFACTORY_PASSWORD} -r ${conan_develop_repo} ${ARTIFACTORY_USER}"
                    sh "conan user -p ${ARTIFACTORY_PASSWORD} -r ${conan_tmp_repo} ${ARTIFACTORY_USER}"
                }
              }
              // create the graph lock for the latest versions of dependencies in develop repo and with the
              // latest revision of the library that trigered this pipeline
              def lockfile = "${profile}.lock"
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
                stage("Launch build: ${product}")
                {
                  // now that we have a lockfile as an input conan install will update the build nodes
                  // doing an install with --build missing would produce the same results:
                  // sh "conan install ${product} --profile ${profile} --lockfile=${lockfile} --build missing "
                  // but using the build order and iterating the list of libraries to build
                  // we could call to the real pipeline that should build the libraries
                  // (here we don't call that pipeline to simplify)
                  stash name: lockfile, includes: lockfile

                  build_order.each { references_list ->
                    def stage_jobs = references_list.each { index_reference ->
                      def lib_name = index_reference[1].split("/")[0]
                      def lib_name_profile = "${lib_name}-${profile}.lock"
                      // here, in the real world, one should invoke the actual lib pipeline, somemthing like:
                      // build(job: "../lib/develop", propagate: true, wait: true, parameters...
                      def upload_ref = (params.library_branch == "develop") ? true : false
                      build_ref_with_lockfile(index_reference[1], lockfile, profile, upload_ref).call()
                      unstash lib_name_profile
                      sh "conan graph update-lock ${lockfile} ${lib_name_profile}"
                      stash name: lockfile, includes: lockfile
                    }
                  }                  

                  lock_contents = readJSON(file: lockfile)
                  sh "cat ${lockfile}"
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

def promote_with_lockfile(lockfile_json, source_repo, target_repo) {
  def references_to_copy = []
  def nodes = lockfile_json['graph_lock'].nodes
  nodes.each { id, node_info ->
    references_to_copy.add(node_info.pref)
  }
  references_to_copy.unique()
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
      // avoid copying again the same package revision
      curl_output = sh (script: "curl -u\"\${ARTIFACTORY_USER}\":\"\${ARTIFACTORY_PASSWORD}\" -XGET \"http://${artifactory_url}:8081/artifactory/api/storage/${target_repo}/${user}/${name_version}/${channel}/${rrev}/package/${pkgid}/${prev}\"", returnStdout: true)
      echo "${curl_output}"
      // check if the path already exists in Artifactory and skip in that case
      if (curl_output.contains("errors")) {
        sh "curl -u\"\${ARTIFACTORY_USER}\":\"\${ARTIFACTORY_PASSWORD}\" -XPOST \"http://${artifactory_url}:8081/artifactory/api/copy/${source_repo}/${user}/${name_version}/${channel}/${rrev}/package/${pkgid}/${prev}?to=${target_repo}/${user}/${name_version}/${channel}/${rrev}/package/${pkgid}/${prev}\""
      }
      else {
        echo "Skipping copy. Location already exists"        
      }
    } 
  }
}

def calc_lockfiles(product, docker_image) {
  return {
    stage("Calc lockfiles") {
      node {
        docker.image(docker_image).inside("--net=host") {
          echo "Inside docker image: ${docker_image}"
          withEnv(["CONAN_USER_HOME=${env.WORKSPACE}/conan_home"]) {
            try {
              stage("Configure conan") {
                sh "conan config install ${config_url}"
                withCredentials([usernamePassword(credentialsId: 'artifactory-credentials', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
                    sh "conan user -p ${ARTIFACTORY_PASSWORD} -r ${conan_develop_repo} ${ARTIFACTORY_USER}"
                    sh "conan user -p ${ARTIFACTORY_PASSWORD} -r ${conan_tmp_repo} ${ARTIFACTORY_USER}"
                }
              }
              // create the graph lock for the latest versions of dependencies in develop repo and with the
              // latest revision of the library that trigered this pipeline
              def name = product.split("/")[0]
              def lockfiles_build_order = [:]
              stage("Install ${params.reference} for all profiles") {
                profiles.each { profile, _ ->
                  sh "conan install ${params.reference} -r ${conan_tmp_repo} --profile ${profile}"
                }
                // we don't want to pick anything from conan-tmp but the new revision of the lib
                // we have just installed
                sh "conan remote remove ${conan_tmp_repo}"
              }
              stage("Calculate lockfiles for building ${name} with ${params.reference}") {
                profiles.each { profile, _ ->
                  echo "calc lockfile for profile: ${profile}"
                  def lockfile = "${name}-${profile}"
                  // install just the recipe for the new revision of the lib and then resolve graph with develop repo
                  // we don't want to bring binaries at this moment, this could be just a agent that
                  // then sends lockfiles to other nodes to build them
                  sh "conan graph lock ${product} --profile=${profile} --lockfile=${lockfile}.lock -r ${conan_develop_repo}"
                  stash name: lockfile, includes: "${lockfile}.lock" 
                  def build_order_file = "${name}-${profile}.json"
                  sh "conan graph build-order ${lockfile}.lock --json=${build_order_file} --build missing"
                  def build_order = readJSON(file: build_order_file)
                  lockfiles_build_order[lockfile] = build_order
                }              
              }
              return lockfiles_build_order
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
    string(name: 'library_branch',)
  }

  stages {
    stage('Build affected products') {
      agent any
      steps {
        script {          
          // show some info
          profiles.each { profile, docker_image ->
              docker.image(docker_image).inside("--net=host") {
                  withEnv(["CONAN_USER_HOME=${env.WORKSPACE}/${profile}/conan_cache/"]) {
                      sh "conan config install ${config_url}"
                      withCredentials([usernamePassword(credentialsId: 'artifactory-credentials', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
                          sh "conan user -p ${ARTIFACTORY_PASSWORD} -r ${conan_develop_repo} ${ARTIFACTORY_USER}"
                          sh "conan user -p ${ARTIFACTORY_PASSWORD} -r ${conan_tmp_repo} ${ARTIFACTORY_USER}"
                      }
                      sh "conan --version"
                      sh "conan config home"                               
                      sh "conan remote list"                               
                  }                      
              }
          }

          // build
          products_build_result = products.collectEntries { product ->
            echo "---> Checking: ${product} <---"            
            def name = product.split("/")[0]
            // App and App2 never should be done in parallel, because you would build some dependencies twice
            stage("Calc lockfiles for ${product}") {
              echo "Calc lockfiles for '${product}'"
              echo " - for changes in '${params.reference}'"
              build_orders = calc_lockfiles(product, "conanio/gcc6").call()
            }

            def product_build_result = [:]

            build_orders.each { lockfile, build_order ->
              echo "----- lockfile ------"
              println lockfile
              echo "----- build_order ------"
              println build_order
              unstash lockfile
              build_order.each { references_list ->
                references_list.each { index_reference ->
                  def lib_name = index_reference[1].split("/")[0]
                  def lib_reference = index_reference[1].split("#")[0]
                  def profile_name = lockfile.split("-",2)[1]
                  echo "-------> build: ${lib_name} <-------"
                  sh "cat ${lockfile}.lock"
                  def lock_contents = readFile(file: "${lockfile}.lock")
                  def build_params = [[$class: 'StringParameterValue', name: "${profile_name}", value: lock_contents]]
                  def build_library_job = build(job: "../${lib_name}/develop", 
                                                propagate: true, wait: true, 
                                                parameters: build_params)
                  def child_build_number = build_library_job.getNumber()
                  def child_job_name = "${lib_name}/develop"
                  def lib_lockfile_path = "/${artifactory_metadata_repo}/${child_job_name}/${child_build_number}/${lib_reference}/${profile_name}/${profile_name}.lock"
                  def base_url = "http://${artifactory_url}:8081/artifactory"
                  withCredentials([usernamePassword(credentialsId: 'artifactory-credentials', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
                      // download
                      sh "curl --user \"\${ARTIFACTORY_USER}\":\"\${ARTIFACTORY_PASSWORD}\" -X GET ${base_url}${lib_lockfile_path} -o modified-${lib_name}-${profile_name}.lock"
                  }                                
                  sh "cat ${lockfile}.lock"
                  sh "cat modified-${lib_name}-${profile_name}.lock"
                  sh "conan graph update-lock ${lockfile}.lock modified-${lib_name}-${profile_name}.lock"
                  sh "cat ${lockfile}.lock"
                  product_build_result["${profile_name}"] = readJSON(file: "${lockfile}.lock")
                }
              }                      
            }
            echo "---------product_build_result------------"
            println product_build_result
            echo "----------------------"
            ["${product}": product_build_result]
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
                withCredentials([usernamePassword(credentialsId: 'artifactory-credentials', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
                    sh "conan user -p ${ARTIFACTORY_PASSWORD} -r ${conan_develop_repo} ${ARTIFACTORY_USER}"
                    sh "conan user -p ${ARTIFACTORY_PASSWORD} -r ${conan_tmp_repo} ${ARTIFACTORY_USER}"
                }

                // promote using each of the lockfiles
                products_build_result.each { product, result ->
                  result.each { profile, lockfile ->
                    if (lockfile.size()>0) {
                      stage("Promote ${profile} binaries to conan-develop") {
                        promote_with_lockfile(lockfile, conan_tmp_repo, conan_develop_repo)
                      }
                      stage("Upload lockfile: ${profile} - ${product}") {
                        def product_name = product.split("/")[0]
                        def lockfile_name = "${product_name}-${profile}.lock"
                        writeJSON file: "${lockfile_name}", json: lockfile
                        def lockfile_path = "/${artifactory_metadata_repo}/${env.JOB_NAME}/${env.BUILD_NUMBER}/${product}/${profile}/${lockfile_name}"
                        def base_url = "http://${artifactory_url}:8081/artifactory"
                        def name = product.split("/")[0]
                        def version = product.split("/")[1].split("@")[0]
                        def properties = "?properties=build.name=${env.JOB_NAME}%7Cbuild.number=${env.BUILD_NUMBER}%7Cprofile=${profile}%7Cname=${name}%7Cversion=${version}"
                        withCredentials([usernamePassword(credentialsId: 'artifactory-credentials', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
                            // upload the lockfile
                            sh "curl --user \"\${ARTIFACTORY_USER}\":\"\${ARTIFACTORY_PASSWORD}\" -X PUT ${base_url}${lockfile_path} -T ${lockfile_name}"
                            // set properties in Artifactory for the file
                            sh "curl --user \"\${ARTIFACTORY_USER}\":\"\${ARTIFACTORY_PASSWORD}\" -X PUT ${base_url}/api/storage${lockfile_path}${properties}"
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
