print "----------------------------------------------------"
print "                 JMeter Testing"
print "----------------------------------------------------"

node('jenkins-agent'){

  workspace = pwd()   // Set the main workspace in your Jenkins agent

  // Set location of OpenShift objects in workspace
  buildConfigPath = "build-config.yaml"
  imageStreamPath = "image-stream.yaml"
  jobTemplatePath = "job-template.yaml"

  project = "devops"    // Set the OpenShift project you're working in
  testSuiteName = "jmeter-test-suite"   // Name of the job/build/imagestream


  sh """
    git clone https://github.com/Vauzc/jmeter-openshift-jenkins.git
    echo 'pwd && ls -l'
  """

  // Create your ImageStream and BuildConfig in OpenShift
  // Then start the build for the test suite image
  sh """
    oc apply -f ${imageStreamPath} -n ${project}
    oc apply -f ${buildConfigPath} -n ${project}
    oc start-build ${testSuiteName} -n ${project} --follow
  """

  // Get latest JMeter image from project to assign to job template
  String imageURL = sh (
    script:"""
      oc get is/rhel-${testSuiteName} -n ${project} --output=jsonpath={.status.dockerImageRepository}
      """,
    returnStdout: true
  )

  // Get all JMX files from JMeter directory
  // Pipeline Utility Steps Plugin --> findFiles
  String files = findFiles(glob: '**/jmeter/*.jmx')

  // Split file names into testFileNames array
  testFileNames = files.split('\n')

  // For every test file, run job and get results back
  for (int i=0; i<testFileNames.size(); i++) {

    // Get file name without .jmx
    file = testFileNames[i]
    fileName = file.replaceAll('.jmx','')

    print "Running JMeter tests: ${fileName}"

    // Set the return URL for the Jenkins input step
    inputURL = env.BUILD_URL + "input/Jmeter-${fileName}/submit"

    // Delete existing test suite job for previous JMX file
    // Create new test suite job with current file
    // Pass in input URL, Jenkins username/password, and image
    sh """
      oc delete job/${testSuiteName} -n ${project} --ignore-not-found=true

      oc process -f ${jobTemplatePath} -p \
      JENKINS_PIPELINE_RETURN_URL=${inputURL} \
      FILE_NAME=${fileName} \
      IMAGE=${imageURL}:latest \
      -n ${project} | oc create -f - -n ${project}
    """

    // Block and wait for job to return with results file
    // The input ID is what creates the unique return URL for curling back
    print "Waiting for results from JMeter test job..."
    def inputFile = input id: "Jmeter-${fileName}",
      message: 'Waiting for JMeter results...',
      parameters: [
        file(description: 'Performance Test Results',
        name: "${fileName}")  // This must match the name param in runjob.sh
      ]

    // Running Performance Plugin with JMeter to show results
    // The results JTL file is saved on the master Jenkins node,
    // inputFile.toString() gives you the absolute path of the file to parse
    //
    // These threshold values can be whatever you like
    performanceReport compareBuildPrevious: false,
      configType: 'ART',
      errorFailedThreshold: 0,
      errorUnstableResponseTimeThreshold: '',
      errorUnstableThreshold: 0,
      failBuildIfNoResultFile: false,
      ignoreFailedBuild: false,
      ignoreUnstableBuild: true,
      modeOfThreshold: false,
      modePerformancePerTestCase: true,
      modeThroughput: true,
      nthBuildNumber: 0,
      parsers: [[$class: 'JMeterParser', glob: inputFile.toString()]],
      relativeFailedThresholdNegative: 0,
      relativeFailedThresholdPositive: 0,
      relativeUnstableThresholdNegative: 0,
      relativeUnstableThresholdPositive: 0
  } // for loop
} // node
