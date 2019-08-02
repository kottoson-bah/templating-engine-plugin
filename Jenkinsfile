node{
  checkout scm
  stash "workspace"
}

stage("Unit Test"){
  node{
    unstash "workspace"

    sh "docker run --rm -v \$(pwd):/home/gradle/project -w /home/gradle/project -u root gradle:4.10.3 gradle clean test && touch \$(ls build/test-results/test/*.xml)"
    archiveArtifacts "target/reports/tests/test/**"
    junit "target/test-results/test/*.xml"
    stash "workspace"
  }
}

parallel "Compile Docs": {
  stage("Compile Docs"){
    unstash "workspace"
    sh "make docs"
    archiveArtifacts "_build/html/**"
  }
}, "Static Code Analysis": {
  stage("Static Code Analysis"){
    echo "placeholder Static Code Analysis code"
  }
}, "Build Plugin" {
  stage("Build Plugin"){
    node{
      unstash "workspace"
      sh "docker run --rm -v \$(pwd):/home/gradle/project -w /home/gradle/project -u root gradle:4.10.3 gradle clean jpi"
      archiveArtifacts "build/libs/templating-engine.hpi"
    }
  }
}
