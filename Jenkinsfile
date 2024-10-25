pipeline 
{ 
  agent any

  options {
        disableConcurrentBuilds()
  }


  environment {
    amiNameTag = ""
    amiNameTagValue = "";
    thisTestNameVar = "";
    thisTestValue = "api-testing";
    ProjectName = "petclinic-spring";
    fileProperties = "file.properties"
  }

  stages {

   stage('Get API Testing Repo')
   {
      steps {
        echo "Getting the API Testing Repo"
        git(
        url:'git@github.com:ochoadevops/petclinic-api.git',
        credentialsId: 'api',
        branch: "main"
        )
     }

   }


   stage('Read Properties File') {
      steps {
        script {

           copyArtifacts(projectName: "${ProjectName}");
           props = readProperties file:"${fileProperties}";

           this_group = props.Group;
           this_version = props.Version;
           this_artifact = props.ArtifactId;
           this_full_build_id = props.FullBuildId;
           this_jenkins_build_id = props.JenkinsBuildId;
        }


        sh "echo Finished setting this_group = $this_group"
        sh "echo Finished setting this_version = $this_version"
        sh "echo Finished setting this_artifact = $this_artifact"
        sh "echo Finished setting this_full_build_id = $this_full_build_id"
        sh "echo Finished setting this_jenkins_build_id = $this_jenkins_build_id"

      }
    }





      stage('Deploying App')
      {
        steps
        {
           echo "Starting --- terraform deploy and start"

           sh 'pwd'
           dir('./infrastructure')
           {
              script {
                 echo "update terraform variables "

                 amiNameTagValue = "$this_artifact" + "-" + "$this_jenkins_build_id";
                 amiNameTag = "build_id=\"" + "$amiNameTagValue" + "\"";
                 thisTestNameVar = "test_name=\"" + "$thisTestValue" + "\"";

                 def readContent = readFile 'terraform.tfvars'
                 writeFile file: 'terraform.tfvars', text: readContent+"\n$amiNameTag"+"\n$thisTestNameVar"

                 sh 'pwd'
                 sh 'ls -l'
                 sh '/usr/local/bin/terraform init -input=false'
                 sh '/usr/local/bin/terraform plan'
                 sh '/usr/local/bin/terraform apply -auto-approve'

                 echo 'Starting sleep for 3 minutes to allow for the EC2 Instance to complete startup'
                 sleep(time: 3, unit: 'MINUTES')
                 echo 'Finished sleep'
              
              }
           }


        }
      }

      stage('update jmeter test')
      {
        steps
        {
           echo "Starting --- update JMeter test plan with new instance ip"

           sh 'pwd'
           dir('./api-test')
           {
           echo "ls -l"
              script {
                 echo "getting new instance ip "

                 def NEW_IP = "";
                 echo "marker-jmeter-01";

                 def FindInstancePublicIP = "";

                 FindInstancePublicIP =  "aws --region us-west-1  ec2 describe-instances --filters \\\'Name=tag:build_id,Values=\\\"TAG_TO_REPLACE\\\"\\\' | grep -i PublicIpAddress | awk '{print \$2 }' | awk '{print substr(\$1,2); }' | awk '{print substr(\$1, 1, length(\$1)-2)}'";

                 echo "Original string is :   ${FindInstancePublicIP} ";

                 echo "updating to: ${amiNameTagValue} ";
                 def badString;
                 def goodString;
                 badString = "\\\\";
                 goodString = "";

                 FindInstancePublicIP =  FindInstancePublicIP.replaceAll("TAG_TO_REPLACE","${amiNameTagValue}");
                 echo "updated string is :   ${FindInstancePublicIP} ";


                 FindInstancePublicIP =  FindInstancePublicIP.replaceAll("${badString}", "${goodString}");
                 echo "final string is :   ${FindInstancePublicIP} ";

                 NEW_IP = sh (returnStdout: true, script: "eval ${FindInstancePublicIP}");
                 echo "New IP is ${NEW_IP}";

                 NEW_IP = NEW_IP.replaceAll("[\r\n]+","");

                 echo "New IP is ${NEW_IP}";
                 echo "";

                 def filenew = readFile('test-plan.jmx').replaceAll("localhost","${NEW_IP}")
                 writeFile file:'./test-plan.jmx', text: filenew

                 echo "Done updating test-plan.jmx";


              }
           }


        }
      }

      stage('Run API Test')
      {
        steps
        {
           echo "Starting --- JMETER API Test"

           sh 'pwd'
           dir('./api-test')
           {
              echo "ls -l"
              script {
                 echo "running test..."
                 sh 'pwd'
                 sh 'ls -l'
                 sh 'rm -f test-results.*'
                 sh 'rm -r -f html-report'
                 sh 'ls -l'
                 sh '/usr/local/bin/jmeter/bin/jmeter.sh -n -t ./test-plan.jmx -l ./test-results.csv'
                 sh 'mkdir html-report'
                 sh 'ls -l'
                 echo "create html report"
                 sh '/usr/local/bin/jmeter/bin/jmeter.sh -g ./test-results.csv -e -o html-report'

                 echo "uploading artifacts to Jenkins dashboard"
                 archiveArtifacts '**/*.*'
                 step([$class: 'ArtifactArchiver', artifacts: '**/*.*'])
              }
           }


        }
      }

      stage('Destroy Environment')
      {
        steps
        {
           dir('./infrastructure')
           {
              script {
                 echo "update terraform variables "
                 // Test completed, destroy environment
                 echo "Test completed, destroying environment"
                 sh '/usr/local/bin/terraform destroy -auto-approve'
              }
          }
      }

      }

  }

 }
