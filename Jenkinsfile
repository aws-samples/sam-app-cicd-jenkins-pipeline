pipeline {
  agent any
  environment {
    PIPELINE_USER_CREDENTIAL_ID = 'aws_creds'
    SAM_TEMPLATE = 'app.yaml'
    MAIN_BRANCH = 'main'
    TESTING_STACK_NAME = 'sam-app'
    TESTING_PIPELINE_EXECUTION_ROLE = 'arn:aws:iam::4876:role/aws-sam-cli-managed-Dev-pipe-PipelineExecutionRole-1IQV0ALGY8E3V'
    TESTING_CLOUDFORMATION_EXECUTION_ROLE = 'arn:aws:iam::4876:role/aws-sam-cli-managed-Dev-p-CloudFormationExecutionR-FJXI1L0E7Z3E'
    TESTING_ARTIFACTS_BUCKET = 'aws-sam-cli-managed-dev-pipeline-artifactsbucket'
    // If there are functions with "Image" PackageType in your template,
    // uncomment the line below and add "--image-repository ${TESTING_IMAGE_REPOSITORY}" to
    // testing "sam package" and "sam deploy" commands.
    // TESTING_IMAGE_REPOSITORY = '0123456789.dkr.ecr.region.amazonaws.com/repository-name'
    TESTING_REGION = 'us-west-2'
    PROD_STACK_NAME = 'sam-app-2'
    PROD_PIPELINE_EXECUTION_ROLE = 'arn:aws:iam::4876:role/aws-sam-cli-managed-Dev-pipe-PipelineExecutionRole-1IQV0ALGY8E3V'
    PROD_CLOUDFORMATION_EXECUTION_ROLE = 'arn:aws:iam::4876:role/aws-sam-cli-managed-Dev-p-CloudFormationExecutionR-FJXI1L0E7Z3E'
    PROD_ARTIFACTS_BUCKET = 'aws-sam-cli-managed-dev-pipeline-artifactsbucket'
    // If there are functions with "Image" PackageType in your template,
    // uncomment the line below and add "--image-repository ${PROD_IMAGE_REPOSITORY}" to
    // prod "sam package" and "sam deploy" commands.
    // PROD_IMAGE_REPOSITORY = '0123456789.dkr.ecr.region.amazonaws.com/repository-name'
    PROD_REGION = 'us-west-2'
  }
  stages {
    // step for running unit-tests
    stage('unit-test') {
        steps {
          sh '''
            pip3 install -r cidr_management/requirements.txt
            export PATH=/var/lib/jenkins/.local/bin:$PATH
            pytest ./ --junitxml=junit_report.xml
            coverage xml -i cidr_management/*
          '''
        }
    }

    // uncomment and modify the following step for running sonarqube-scan
    /* stage('static-code-analysis'){
       steps {
        withSonarQubeEnv('SonarQubeEC2Server') {
            sh '''
            sonar-scanner-4.6.2.2472-linux/bin/sonar-scanner
            '''
        }
      }
     } */

    stage('build-and-deploy-feature') {
      // this stage is triggered only for feature branches (feature*),
      // which will build the stack and deploy to a stack named with branch name.
      when {
        branch 'feature*'
      }
      agent {
        docker {
          image 'public.ecr.aws/sam/build-provided'
          args '--user 0:0 -v /var/run/docker.sock:/var/run/docker.sock'
        }
      }
      steps {
        sh 'sam build --template ${SAM_TEMPLATE} --use-container'
        withAWS(
            credentials: env.PIPELINE_USER_CREDENTIAL_ID,
            region: env.TESTING_REGION,
            role: env.TESTING_PIPELINE_EXECUTION_ROLE,
            roleSessionName: 'deploying-feature') {
          sh '''
            sam deploy --stack-name $(echo ${BRANCH_NAME} | tr -cd '[a-zA-Z0-9-]') \
              --capabilities CAPABILITY_IAM \
              --region ${TESTING_REGION} \
              --s3-bucket ${TESTING_ARTIFACTS_BUCKET} \
              --no-fail-on-empty-changeset \
              --role-arn ${TESTING_CLOUDFORMATION_EXECUTION_ROLE}
          '''
        }
      }
    }

    stage('build-and-package') {
      when {
        branch env.MAIN_BRANCH
      }
      agent {
        docker {
          image 'public.ecr.aws/sam/build-provided'
          args '--user 0:0 -v /var/run/docker.sock:/var/run/docker.sock'
        }
      }
      steps {
        sh 'sam build --template ${SAM_TEMPLATE} --use-container'
        withAWS(
            credentials: env.PIPELINE_USER_CREDENTIAL_ID,
            region: env.TESTING_REGION,
            role: env.TESTING_PIPELINE_EXECUTION_ROLE,
            roleSessionName: 'testing-packaging') {
          sh '''
            sam package \
              --s3-bucket ${TESTING_ARTIFACTS_BUCKET} \
              --region ${TESTING_REGION} \
              --output-template-file packaged-testing.yaml
          '''
        }
        archiveArtifacts artifacts: 'packaged-testing.yaml'

      }
    }

    stage('deploy-to-testing') {
      when {
        branch env.MAIN_BRANCH
      }
      agent {
        docker {
          image 'public.ecr.aws/sam/build-provided'
          args '--user 0:0 -v /var/run/docker.sock:/var/run/docker.sock'
        }
      }
      steps {
        withAWS(
            credentials: env.PIPELINE_USER_CREDENTIAL_ID,
            region: env.TESTING_REGION,
            role: env.TESTING_PIPELINE_EXECUTION_ROLE,
            roleSessionName: 'testing-deployment') {
          sh '''
            sam deploy --stack-name ${TESTING_STACK_NAME} \
              --template packaged-testing.yaml \
              --capabilities CAPABILITY_IAM \
              --region ${TESTING_REGION} \
              --s3-bucket ${TESTING_ARTIFACTS_BUCKET} \
              --no-fail-on-empty-changeset \
              --role-arn ${TESTING_CLOUDFORMATION_EXECUTION_ROLE}
          '''
        }
      }
    }

    // step for running the integration-tests
     stage('integration-test') {
       when {
         branch env.MAIN_BRANCH
       }
       steps {
         withAWS(
            credentials: env.PIPELINE_USER_CREDENTIAL_ID,
            region: env.TESTING_REGION,
            role: env.TESTING_PIPELINE_EXECUTION_ROLE,
            roleSessionName: 'testing-deployment') {
          sh '''
           export PATH=/var/lib/jenkins/.local/bin:$PATH
           behave cidr_management/integ_test/
         '''
        }
       }
     }
  }
}
