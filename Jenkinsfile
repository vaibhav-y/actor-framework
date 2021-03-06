#!/usr/bin/env groovy

// List of CMake arguments used by most builds.
defaultCmakeArgs = '-DCAF_MORE_WARNINGS:BOOL=yes' \
                   + ' -DCAF_ENABLE_RUNTIME_CHECKS:BOOL=yes' \
                   + ' -DOPENSSL_ROOT_DIR=/usr/local/opt/openssl' \
                   + ' -DOPENSSL_INCLUDE_DIR=/usr/local/opt/openssl/include' \

// Our build matrix. The keys are the operating system labels and the values
// are lists of tool labels.
buildMatrix = [
  // Debug builds with ASAN + logging for various OS/compiler combinations.
  ['unix', [
    builds: ['debug'],
    tools: ['gcc4.8', 'gcc4.9', 'gcc5', 'gcc6', 'gcc7', 'gcc8', 'clang'],
    cmakeArgs: defaultCmakeArgs
               + ' -DCAF_LOG_LEVEL:STRING=4'
               + ' -DCAF_ENABLE_ADDRESS_SANITIZER:BOOL=yes',
  ]],
  // One release build per supported OS. FreeBSD and Windows have the least
  // testing outside Jenkins, so we also explicitly schedule debug builds.
  ['Linux', [
    builds: ['release'],
    tools: ['gcc', 'clang'],
    cmakeArgs: defaultCmakeArgs,
  ]],
  ['macOS', [
    builds: ['release'],
    tools: ['clang'],
    cmakeArgs: defaultCmakeArgs,
  ]],
  ['FreeBSD', [
    builds: ['debug'], // no release build for now, because it takes 1h
    tools: ['clang'],
    cmakeArgs: defaultCmakeArgs,
  ]],
  ['Windows', [
    builds: ['debug', 'release'],
    tools: ['msvc'],
    cmakeArgs: '-DCAF_BUILD_STATIC_ONLY:BOOL=yes'
               + ' -DCAF_ENABLE_RUNTIME_CHECKS:BOOL=yes'
               + ' -DCAF_NO_OPENCL:BOOL=yes',
  ]],
  // One Additional build for coverage reports.
  ['unix', [
    builds: ['debug'],
    tools: ['gcovr'],
    extraSteps: ['coverageReport'],
    cmakeArgs: '-DCAF_ENABLE_GCOV:BOOL=yes '
               + ' -DCAF_NO_EXCEPTIONS:BOOL=yes '
               + ' -DCAF_FORCE_NO_EXCEPTIONS:BOOL=yes',
  ]],
]

// Optional environment variables for combinations of labels.
buildEnvironments = [
  'macOS && gcc': ['CXX=g++'],
  'Linux && clang': ['CXX=clang++'],
]

// Called *after* a build succeeded.
def coverageReport() {
  dir('caf-sources') {
    sh 'gcovr -e examples -e tools -e libcaf_test -e ".*/test/.*" -e libcaf_core/caf/scheduler/profiled_coordinator.hpp -x -r .. > coverage.xml'
    archiveArtifacts '**/coverage.xml'
    cobertura([
      autoUpdateHealth: false,
      autoUpdateStability: false,
      coberturaReportFile: '**/coverage.xml',
      conditionalCoverageTargets: '70, 0, 0',
      failUnhealthy: false,
      failUnstable: false,
      lineCoverageTargets: '80, 0, 0',
      maxNumberOfBuilds: 0,
      methodCoverageTargets: '80, 0, 0',
      onlyStable: false,
      sourceEncoding: 'ASCII',
      zoomCoverageChart: false,
    ])
  }
}

def cmakeSteps(buildType, cmakeArgs) {
  // Configure and build.
  cmakeBuild([
    buildDir: 'build',
    buildType: buildType,
    cmakeArgs: cmakeArgs,
    installation: 'cmake in search path',
    sourceDir: '.',
    steps: [[withCmake: true]],
  ])
  // Run unit tests.
  ctest([
    arguments: '--output-on-failure',
    installation: 'cmake in search path',
    workingDir: 'build',
  ])
}

def buildSteps(buildType, cmakeArgs) {
  echo "build stage: $STAGE_NAME"
  deleteDir()
  unstash('caf-sources')
  dir('caf-sources') {
    if (STAGE_NAME.contains('Windows')) {
      echo "Windows build on $NODE_NAME"
      withEnv(['PATH=C:\\Windows\\System32;C:\\Program Files\\CMake\\bin;C:\\Program Files\\Git\\cmd;C:\\Program Files\\Git\\bin']) {
        cmakeSteps(buildType, cmakeArgs)
      }
    } else {
      echo "Unix build on $NODE_NAME"
      def leakCheck = STAGE_NAME.contains("Linux") && !STAGE_NAME.contains("clang")
      withEnv(["label_exp=" + STAGE_NAME.toLowerCase(),
               "ASAN_OPTIONS=detect_leaks=" + (leakCheck ? 1 : 0)]) {
        cmakeSteps(buildType, cmakeArgs)
      }
    }
  }
}

// Builds a stage for given builds. Results in a parallel stage `if builds.size() > 1`.
def makeBuildStages(matrixIndex, builds, lblExpr, settings) {
  builds.collectEntries { buildType ->
    def id = "$matrixIndex $lblExpr: $buildType"
    [
      (id):
      {
        node(lblExpr) {
          stage(id) {
            try {
              withEnv(buildEnvironments[lblExpr] ?: []) {
                buildSteps(buildType, settings['cmakeArgs'] ?: '')
                (settings['extraSteps'] ?: []).each { fun -> "$fun"() }
              }
            } finally {
              cleanWs()
            }
          }
        }
      }
    ]
  }
}

pipeline {
  agent none
  environment {
    LD_LIBRARY_PATH = "$WORKSPACE/caf-sources/build/lib"
    DYLD_LIBRARY_PATH = "$WORKSPACE/caf-sources/build/lib"
  }
  stages {
    stage ('Git Checkout') {
      agent { label 'master' }
      steps {
        deleteDir()
        dir('caf-sources') {
          checkout scm
        }
        stash includes: 'caf-sources/**', name: 'caf-sources'
      }
    }
    // Start builds.
    stage('Builds') {
      steps {
        script {
          // Create stages for building everything in our build matrix in
          // parallel.
          def xs = [:]
          buildMatrix.eachWithIndex { entry, index ->
              def (os, settings) = entry
              settings['tools'].eachWithIndex { tool, toolIndex ->
                def matrixIndex = "[$index:$toolIndex]"
                def builds = settings['builds']
                def labelExpr = "$os && $tool"
                xs << makeBuildStages(matrixIndex, builds, labelExpr, settings)
              }
          }
          parallel xs
        }
      }
    }
  }
  post {
    success {
      emailext(
        subject: "✅ CAF build #${env.BUILD_NUMBER} succeeded for branch ${env.GIT_BRANCH}",
        recipientProviders: [culprits(), developers(), requestor(), upstreamDevelopers()],
        body: "Job ${env.JOB_NAME} [${env.BUILD_NUMBER}] passed.\n\n" \
              + "Check console output at ${env.BUILD_URL}.",
      )
    }
    failure {
      emailext(
        subject: "⛔️ CAF build #${env.BUILD_NUMBER} failed for branch ${env.GIT_BRANCH}",
        attachLog: true,
        compressLog: true,
        recipientProviders: [culprits(), developers(), requestor(), upstreamDevelopers()],
        body: "Job ${env.JOB_NAME} [${env.BUILD_NUMBER}] failed!\n\n" \
              + "Check console output at ${env.BUILD_URL} or see attached log.",
      )
    }
  }
}

