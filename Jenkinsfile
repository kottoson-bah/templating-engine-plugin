node{
  checkout scm
  stash "workspace"
}

stage("Unit Test"){
  node{
    unstash "workspace"

    sh "docker run --rm -v \$(pwd):/home/gradle/project -w /home/gradle/project -u root gradle:4.10.3 gradle clean test && touch \$(ls build/test-results/test/*.xml)"
    archiveArtifacts "build/reports/tests/test/**"
    junit "build/test-results/test/*.xml"
    stash "workspace"
  }
}

parallel "Compile Docs": {
  stage("Compile Docs"){
    node{
      unstash "workspace"
      sh "make docs"
      archiveArtifacts "docs/_build/html/**"
    }
  }
}, "Static Code Analysis": {
  stage("Static Code Analysis"){
    static_code_analysis()
  }
}, "Build Plugin": {
  stage("Build Plugin"){
    node{
      unstash "workspace"
      sh "docker run --rm -v \$(pwd):/home/gradle/project -w /home/gradle/project -u root gradle:4.10.3 gradle clean jpi"
      archiveArtifacts "build/libs/templating-engine.hpi"
    }
  }
}


def static_code_analysis(){

  // sonarqube api token
  cred_id = config.credential_id ?:
            "sonarqube"

  enforce = config.enforce_quality_gate ?:
            true

  stage("SonarQube Analysis"){
    inside_sdp_image "sonar-scanner", {
      withCredentials([usernamePassword(credentialsId: cred_id, passwordVariable: 'token', usernameVariable: 'user')]) {
        withSonarQubeEnv("SonarQube"){
          unstash "workspace"
          try{ unstash "test-results" }catch(ex){}
          sh "mkdir -p empty"
          projectKey = "$env.REPO_NAME:$env.BRANCH_NAME".replaceAll("/", "_")
          projectName = "$env.REPO_NAME - $env.BRANCH_NAME"
          def script = """sonar-scanner -X -Dsonar.login=${user} -Dsonar.password=${token} -Dsonar.projectKey="$projectKey" -Dsonar.projectName="$projectName" -Dsonar.projectBaseDir=. """

          if (!fileExists("sonar-project.properties"))
            script += "-Dsonar.sources=\"./src\""

          sh script

        }
        timeout(time: 1, unit: 'HOURS') {
          def qg = waitForQualityGate()
          if (qg.status != 'OK' && enforce) {
            error "Pipeline aborted due to quality gate failure: ${qg.status}"
          }
        }
      }
    }
  }
}

void inside_sdp_image(String img, Closure body){

  config.images ?: { error "SDP Image Config not defined in Pipeline Config" } ()

  def sdp_img_reg = 'docker-registry.default.svc:5000'

  def sdp_img_repo = "pipeline-for-sdp"

  def sdp_img_repo_cred = 'openshift-docker-registry'

  def docker_args = ""

  docker.withRegistry(sdp_img_reg, sdp_img_repo_cred){
    docker.image("${sdp_img_repo}/${img}").inside("${docker_args}"){
      body.resolveStrategy = Closure.DELEGATE_FIRST
      body.delegate = this
      body()
    }
  }
}
