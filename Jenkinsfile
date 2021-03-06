/////////////////[ Pipeline Configuration ]//////////////////
def git_url = 'git@github.wdf.sap.corp:/Jam-clm/login_proxy.git'

def git_spec = [branch: BRANCH_NAME, credentialsId: 'github-clm', url: git_url]

def aws_bases() { return [
  'us-west-1': '371089343861.dkr.ecr.us-west-1.amazonaws.com',
  'eu-central-1': '371089343861.dkr.ecr.eu-central-1.amazonaws.com',
]}

def aws_k8s_namespaces() { return [
  'us-west-1' : 'kora-test',
  'eu-central-1' : 'eu-central-prod',
]}

def aws_repository() { return 'kora/login-proxy' }

def local_registry() { return 'clm-registry.mo.sap.corp:5000' }

def image_base() { return 'kora/login-proxy' }

def local_images = [
  prod: image_base(),
  dev: image_base() + '-dev',
  build: image_base() + '-build',
]

def container_prompts = [
  dev: 'loginproxy-dev',
  test: 'loginproxy-test',
  prod_prefix: 'loginproxy-'
]

def kubernetes = [
  deployment: 'deploy/loginproxy',
  pod_image: 'loginproxy'
]

def mail_recipients = 'DL SAP Jam CLM <DL_58AF12A15F99B7D3BC000054@exchange.sap.corp>'
/////////////////[ End Pipeline Configuration ]//////////////////

pipeline {
  agent {
    label "meta-only"
  }
  stages{
    stage("Checkout & build") {
      agent {
          label "docker"
      }
      steps {
        step([$class:'WsCleanup'])
        git git_spec
        script {
          currentBuild.description = generate_tag()
          docker_image_cleanup()

          dev_image_stable   = local_repo_image(local_images.dev, stable_tag())
          prod_image_stable  = local_repo_image(local_images.prod, stable_tag())
          build_image_stable = local_repo_image(local_images.build, stable_tag())

          dev_image_tagged   = local_repo_image(local_images.dev, currentBuild.description)
          prod_image_tagged  = local_repo_image(local_images.prod, currentBuild.description)
          build_image_tagged = local_repo_image(local_images.build, currentBuild.description)

          docker_pull(   dev_image_stable, true )
          docker_pull(  prod_image_stable, true )
          docker_pull( build_image_stable, true )

          sh "docker/jenkins-build.sh dev ${dev_image_tagged}" + \
                                    " --prompt ${container_prompts.dev}" + \
                                    " --build-info build=${currentBuild.description}"

          sh "docker/jenkins-build.sh prod ${prod_image_tagged}" + \
                                    " --build ${build_image_tagged}" + \
                                    " --prompt ${container_prompts.prod_prefix}${currentBuild.description}" + \
                                    " --build-info build=${currentBuild.description}"

          // never push latest, since this build hasn't been validated

          // these image names contain the build/git-sha qualified tags (ie, we're not pushing anything as stable)
          docker_push(   dev_image_tagged )
          docker_push(  prod_image_tagged )
          docker_push( build_image_tagged )

          stash name: "docker_config", includes: "docker/*"
        }
      }
    }
    stage("Test") {
      parallel {
        stage("Code Tests (Elixir)") {
          agent{
              label "docker"
          }
          steps {
            step([$class:'WsCleanup'])
            script {
              unstash "docker_config"
              docker_image_cleanup()

              dev_image_tagged = local_repo_image(local_images.dev, currentBuild.description)
              tag_spec = 'TAG=' + currentBuild.description + ' '

              docker_pull( dev_image_tagged )
              docker_pull( 'redis:3.0' )

              // get these set up before docker manually constructs them as root owned
              sh('mkdir -p reports cover testcases')
              sh('chmod a+w reports cover testcases')

              sh('docker-compose -f docker/jenkins.yml down --remove-orphans')

              //
              // Prepare the environment (bring up db and run migrations)
              //
              sh(tag_spec + 'docker-compose -f docker/jenkins.yml up -d redis')
              sleep(10) // 10s

              //
              // Run the actual tests
              //
              try {

                sh(tag_spec + 'docker-compose -f docker/jenkins.yml run --rm service_test mix coveralls.html')

              } finally {
                sh(tag_spec + 'docker-compose -f docker/jenkins.yml down')
                sleep(10) // given docker compose a chance to shut everything down before generating reports and invoking GC
                generate_reports()
              }
            }
          }
        }
        stage("API Tests") {
          agent{
              label "docker"
          }
          steps {
            step([$class:'WsCleanup'])
            script {
              unstash "docker_config"
              docker_image_cleanup()

              api_test_image = local_repo_image('clm-api-testing', '')
              prod_image_tagged = local_repo_image(local_images.prod, currentBuild.description)
              dev_image_tagged = local_repo_image(local_images.dev, currentBuild.description)
              tag_spec = 'TAG=' + currentBuild.description + ' '

              docker_pull( prod_image_tagged )
              docker_pull( dev_image_tagged )
              docker_pull( 'redis:3.0' )
              docker_pull( api_test_image )

              // get these set up before docker manually constructs them as root owned
              sh('mkdir -p reports cover testcases')
              sh('chmod a+w reports cover testcases')

              sh('docker-compose -f docker/jenkins.yml down --remove-orphans')

              //
              // Prepare the environment (bring up db & prod instance)
              //
              sh(tag_spec + 'docker-compose -f docker/jenkins.yml up -d redis')
              sleep(10) // 10s
              sh(tag_spec + 'docker-compose -f docker/jenkins.yml up -d service_prod')
              sleep(10) // 10s

                // from the container, figure out the ip/port of the service_prod
              service_domain = sh(returnStdout: true, script: tag_spec + 'docker-compose -f docker/jenkins.yml run --rm ' + \
                'service_test env | grep -i service_prod_port= | cut -d = -f 2'
              ).trim().replace( "tcp://", "http://" )
              //
              // Run the actual tests: populate the db via api and then run the test runner
              //
              try {

                // service_test implicitly knows location of service_prod via env variable
                // this should be updated to be declarative
                sh(tag_spec + "docker-compose -f docker/jenkins.yml run --rm -e MIX_ENV=dev service_test mix data.populate --api --service ${service_domain}")

                sh(tag_spec + 'docker-compose -f docker/jenkins.yml run --rm api_test_runner')

              } finally {
                sh(tag_spec + 'docker-compose -f docker/jenkins.yml down')
                sleep(10) // given docker compose a chance to shut everything down before generating reports and invoking GC
                archive_xml_report()
              }
            }
          }

        }
      }
    }
    stage("Publish") {
      agent {
          label "docker"
      }
      steps {
        script {
          docker_image_cleanup()

          //
          // pull our build-specific images to the local machine
          //
          dev_image_tagged   = local_repo_image(local_images.dev,   currentBuild.description)
          prod_image_tagged  = local_repo_image(local_images.prod,   currentBuild.description)
          build_image_tagged = local_repo_image(local_images.build, currentBuild.description)

          docker_pull( dev_image_tagged )
          docker_pull( prod_image_tagged )
          docker_pull( build_image_tagged )

          //
          // since this particular build is a success, tag it as stable
          // then push to our local dev registry
          //
          dev_image_stable = local_repo_image(local_images.dev, stable_tag())
          prod_image_stable = local_repo_image(local_images.prod, stable_tag())
          build_image_stable = local_repo_image(local_images.build, stable_tag())

          docker_tag(   dev_image_tagged, dev_image_stable )
          docker_tag(  prod_image_tagged, prod_image_stable )
          docker_tag( build_image_tagged, build_image_stable )

          docker_push(   dev_image_stable )
          docker_push(  prod_image_stable )
          docker_push( build_image_stable )

          //
          // finally, tag our images for pushing to aws (build-tagged & stable)
          // then login to aws & push the images up
          //   ... for each aws region/container registry
          //
          aws_bases().each { region, repo_base ->

            aws_image_tagged = aws_repo_image(repo_base, currentBuild.description)
            aws_image_stable = aws_repo_image(repo_base, stable_tag())

            docker_tag( prod_image_tagged, aws_image_tagged )
            docker_tag( prod_image_tagged, aws_image_stable )

            // login to aws
            withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'aws-ec2-access',
              usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY']]) {

              sh('$( aws ecr get-login --no-include-email --region ' + region + ' )')
            }

            docker_push( aws_image_tagged )
            docker_push( aws_image_stable )

          }
        }
      }
    }
    stage("Deploy") {
      agent {
          label "docker"
      }
      steps {
        script {
          //
          // Assuming the previous step succeeded, we should have the successful docker image in aws already
          // Now we just use kubectl to update the yaml, specifying the new image should be used in the deployment
          //  it should automatically trigger a rolling update
          //
          //   ... for each aws region/container registry

          aws_bases().each { region, repo_base ->
            aws_image_tagged = aws_repo_image(repo_base, currentBuild.description)
            aws_image_stable = aws_repo_image(repo_base, stable_tag())

            credentials_id = 'k8s-access-' + region
            namespace = aws_k8s_namespaces()[region]

            withCredentials([file(credentialsId: credentials_id, variable: 'KUBECONFIGFILE')]) {
              sh('kubectl --kubeconfig=$KUBECONFIGFILE -n ' + namespace + ' set image ' + kubernetes.deployment + ' ' + kubernetes.pod_image + '=' + aws_image_tagged)
            }

          }
        }
      }
    }
  }
  post {
    always {
      step([$class: 'Mailer', notifyEveryUnstableBuild: true, recipients: mail_recipients, sendToIndividuals: true])
    }
  }

}

//
// Reporting
//

def generate_reports() {
    archive_xml_report()
    // Displays coverage reports
    archive (includes: 'cover/*.html')
    publishHTML (target: [
      allowMissing: false,
      keepAll: false,
      reportDir: 'cover',
      reportFiles: 'excoveralls.html',
      reportName: "Coverage Report"
    ])
}

def archive_xml_report() {
    // Displays xml test report
    step([$class: 'JUnitResultArchiver', testResults: 'reports/*.xml'])
}

def send_mail(mail_recipients) {
  step([$class: 'Mailer', notifyEveryUnstableBuild: true, recipients: mail_recipients, sendToIndividuals: true])
}


//
// Helper functions
//

//
// generate_tag (for tagging both build and docker image result)
//
// 12-fcc5e800
// 13-4e5a6be4-mybranch
// branch is only specified if it is not master
//
def generate_tag() {
    buildNumber = currentBuild.number
    gitCommit = sh(returnStdout: true, script: 'git rev-parse --short --verify HEAD').trim()
    branch = (BRANCH_NAME != "master") ? "-" + BRANCH_NAME : ""
    return buildNumber + '-' + gitCommit + branch
}
def stable_tag() {
    branch = (BRANCH_NAME != "master") ? "stable-" + BRANCH_NAME : "stable"
    return branch
}

//
// local_repo_image & aws_repo_image
//  (image name for the local image repository & aws)
// local takes the sub-image, (ie, dev/test/prod)
//
def local_repo_image(image_mode, tag) {
    if (tag == '') {
      return local_registry() + '/' + image_mode
    } else {
      return local_registry() + '/' + image_mode + ':' + tag
    }
}

def aws_repo_image(base, tag) {
    return base + '/' + aws_repository() + ':' + tag;
}

// docker pull
def docker_pull(image, pass_if_image_missing = false) {
    sh 'docker pull ' + image + (pass_if_image_missing ? ' || true' : '')
}
def docker_push(image) {
    sh 'docker push ' + image
}
def docker_image_cleanup() {
    sh '(docker volume ls -qf dangling=true | xargs -r docker volume rm) || true'
    sh '(docker images --filter dangling=true | awk \'NR>1{ print $3 }\' | xargs -r docker rmi) || true'
}
def docker_tag(image1, image2) {
    sh('docker tag ' + image1 + ' ' + image2)
}

