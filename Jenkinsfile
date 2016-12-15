#!/usr/bin/env groovy
registry_url = "https://index.docker.io/v1/"
docker_creds_id = "rlewkowicz"
build_tag = "testing"
maintainer_name = "rlewkowicz"
container_name = "restbase"
node{
  retry( count: 3  ){
    timeout(time: 60, unit: 'SECONDS') {
      stage('Pull and Update') {
        git url: 'https://github.com/rlewkowicz/docker-mediawiki-restbase.git'
        sh './update.sh'
      }
    }
  }
  stage('Build') {
    echo "Building restbase with docker.build(${maintainer_name}/${container_name}:${build_tag})"
    container = docker.build("${maintainer_name}/${container_name}:${build_tag}", '6.9/alpine/')
  }
  stage('Test') {
    try {

      docker.image("${maintainer_name}/${container_name}:${build_tag}").withRun("--name=${container_name} -d -p 127.0.0.1:7231:7231")  { c ->
        timeout(time: 60, unit: 'SECONDS'){
           waitUntil {
               def wait = sh([script: $/docker logs ${container_name} | grep "Startup finished"/$, returnStatus: true])
               return (wait == 0);
           }
         }

         MAX_TESTS = 2
         for (test_num = 0; test_num < MAX_TESTS; test_num++) {

             echo "Running Test(${test_num})"

             expected_results = 0
             if (test_num == 0 )
             {
                 test_results = sh([script: "curl -s 127.0.0.1:8500 | grep -o 'Welcome.*Parsoid'", returnStatus:true])
                 build_tag = sh([script: $/curl -s https://www.npmjs.com/package/restbase | grep strong | grep -o "[0-9]*\.[0-9]*\.[0-9]*"/$, returnStdout: true])
                 if (test_results != 0){
                   currentBuild.result = 'FAILURE'
                   error "Failed to finish container testing. Parsoid Not running"
                 }
             }
             else if (test_num == 1)
             {
                 // Test that port 80 is exposed
                 echo "Exposed Docker Ports:"
                 test_results = sh([script: "docker inspect --format '{{ (.NetworkSettings.Ports) }}' ${container_name} | grep map | grep '8000/tcp:'", returnStatus:true])
                 if (test_results != 0){
                   currentBuild.result = 'FAILURE'
                   error "Failed to finish container testing. Ports not exposed"
                 }
             }
             else
             {
                 err_msg = "Missing Test(${test_num})"
                 echo "ERROR: ${err_msg}"
                 currentBuild.result = 'FAILURE'
                 error "Failed to finish container testing with Message(${err_msg})"
             }
         }
      }

      } catch (Exception err) {
        err_msg = "Test had Exception(${err})"
        currentBuild.result = 'FAILURE'
        error "FAILED - Stopping build for Error(${err_msg})"
      }
  }
  stage('Tag & Upload') {
    echo "Current Build ${build_tag}"
    container.tag([build_tag])
    container.push([build_tag])
    container.tag(['latest'])
    container.push(['latest'])
    echo "Pushed Build ${build_tag}"
    currentBuild.result = 'SUCCESS'
  }
}
