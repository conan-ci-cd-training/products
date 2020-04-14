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
              stage("Insert the new revision ${params.reference} in ${product} graph") {
                // this is a workaround, because installing with a specific reference does not create a lockfile
                // and also, we need the information of the build nodes
                // sh "conan graph lock ${product}  --profile ${profile} --lockfile=${lockfile} -r ${conan_develop_repo}"
                // develop should be consistent without missing packages
                sh "conan install ${product} --profile ${profile} -r ${conan_develop_repo}"
                // now install the newly created revision of the library with -- update
                sh "conan install ${params.reference} --profile ${profile} -r ${conan_tmp_repo} --update"
                // now the cache is populated with exactly the packages we want
                sh "conan graph lock ${product} --profile ${profile} --lockfile=${lockfile}"
                // now that we have a lockfile as an input conan install will update the build nodes
                sh "conan install ${product} --profile ${profile} --lockfile=${lockfile} --build missing "
                sh "cat ${lockfile}"
              }
              stage("Upload") {
                // we upload to tmp, and later if everything is OK will promote to develop
                sh "conan upload '*' --all -r ${conan_tmp_repo} --confirm  --force"
              }
              stage("Read lockfile") {
                  lock_contents = readJSON(file: lockfile)
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

    stage("Check affected products") {
      steps {
        script {
          // get the first element of the profiles list
          // this is just to have a valid profile to use for generating the lockfile
          // that we will use to get the build-order and check if the products are affected
          // by the newly created package revision (reference param)
          String profile = profiles.keySet().iterator().next()
          String docker_image = profiles.get(profile)
          docker.image(docker_image).inside("--net=host") {
            withEnv(["CONAN_USER_HOME=${env.WORKSPACE}/conan_cache"]) {
              try {
                sh "conan config install ${config_url}"
                sh "conan remote add ${conan_develop_repo} http://${artifactory_url}:8081/artifactory/api/conan/${conan_develop_repo}" // the namme of the repo is the same that the arttifactory key
                withCredentials([usernamePassword(credentialsId: 'artifactory-credentials', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
                  sh "conan user -p ${ARTIFACTORY_PASSWORD} -r ${conan_develop_repo} ${ARTIFACTORY_USER}"
                }
                products.each { product ->
                  println "name: ${product.key} repo: ${product.value}"
                  def lockfile = "${product.key}.lock"
                  def bo_file = "${product.key}.json"
                  sh "conan graph lock ${product.key} --profile ${profile} --lockfile=${lockfile} -r ${conan_develop_repo}"
                  sh "conan graph build-order ${lockfile} --json=${bo_file} --build"
                  String reference_name = "${params.reference.split("#")[0]}"
                  build_order = readJSON file: bo_file
                  // nested list
                  build_order.each { libs ->
                    libs.each { lib ->
                      String require = "${lib[1]}"
                      println "checking if ${require} has ${reference_name} --> affects product ${product.key}"
                      if (require.contains("${reference_name}")) {
                        affected_products.add(product.key)
                        println "added ${product.key} to affected products"
                      }
                    }
                  }
                }
                println "Affected products:"
                affected_products.each { prod ->
                  println("${prod}")
                }
                println "will be built"
              }
              finally {
                  deleteDir()
              }
            }
          }
        }
      }
    }

    stage('Build affected products') {
      agent any
      steps {
        script {          
          products_build_result = affected_products.collectEntries { product ->
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

    stage("Promote") {
      steps {
        script {
          docker.image("conanio/gcc8").inside("--net=host") {
            // promote libraries to develop
            if (library_branch == "develop") {       
              withEnv(["CONAN_USER_HOME=${env.WORKSPACE}/conan_cache"]) {
                try {
                  sh "conan config install ${config_url}"
                  sh "conan remote add ${conan_develop_repo} http://${artifactory_url}:8081/artifactory/api/conan/${conan_develop_repo}" // the namme of the repo is the same that the arttifactory key
                  sh "conan remote add ${conan_tmp_repo} http://${artifactory_url}:8081/artifactory/api/conan/${conan_tmp_repo}" // the namme of the repo is the same that the arttifactory key
                  withCredentials([usernamePassword(credentialsId: 'artifactory-credentials', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
                      sh "conan user -p ${ARTIFACTORY_PASSWORD} -r ${conan_develop_repo} ${ARTIFACTORY_USER}"
                      sh "conan user -p ${ARTIFACTORY_PASSWORD} -r ${conan_tmp_repo} ${ARTIFACTORY_USER}"
                  }
                  products_build_result.each { product, result ->
                    result.each { profile, lockfile ->
                      writeJSON file: "${profile}.lock", json: lockfile
                      sh "cat ${profile}.lock"
                      sh "conan install ${product} --lockfile=${profile}.lock"
                    }
                  }
                  sh "conan upload '*' --all -r ${conan_develop_repo} --confirm  --force"
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
