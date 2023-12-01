#!/usr/bin/groovy
@Library('swb-workflow') _

buildWorkflow([
  helpers: [
  
   bitbucket: [:],
   hodor: [type: "app", artifact:"com.amadeus.hodor:hodor:master@latest", path: "bin/hodor",
              appCredentials: "umesh_playwright", withCredentials: true]
  ],
  build: [
    'Override versions': { h ->
      catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
        h.hodor "override-npm-versions ${env.WORKSPACE}/package.json ${env.BRANCH_NAME} -c ${env.CHANGE_URL}"
      }
    },
    'Build': { h -> h.npm 'ci' },
    'Lint': {h -> h.npm 'run lint'},
    'Coverage': {h ->
      h.npm "run coverage"
      def testCoverageJson = readJSON file: "results.json"
      h.bitbucket.addCommentOnPullRequest { "## Test Coverage Report\n ${generateTestCoverageReport(testCoverageJson)}" }
    },
    'Run tests': {h ->
      try {
        h.npm 'run test:build'
      } catch (e) {
        zip zipFile: 'playwright-report.zip', archive: true, dir: 'playwright-report'
      }
    },
    'Display coverage': { h ->
      def results = readJSON file: "results.json"
      h.bitbucket.addCommentOnPullRequest{"## Test run report\n ${markdownTestResults(results)}"}
    }
  ]
])


String markdownTestResults(results) {
  StringBuilder sb = new StringBuilder()
  sb.append('\n| Test name | Test Status |\n')
  sb.append('|  --- | --- | \n')

  results.suites.each { suite ->
    suite.specs.each { spec ->
      if (spec.ok) {
        sb.append("| ${spec.title} | :white_check_mark:. | \n")
      } else {
        sb.append("| ${spec.title} | :x:. | \n")
      }
    }
  }
  return sb.toString()
}

/**
 * Method to prepare a markdown table for functional area's spec and test statistics
 * @param sb : string builder instance
 * @param funcSpecData: Functional area collective spec data
 * @param funcTestData: Functional area collective test block data
 */
private void prepareFunctionalTable(StringBuilder sb, Map<String, Set> funcSpecData, Map<String, Integer> funcTestData) {
    sb.append('| Functional Area Statistics|||\n')
    sb.append('|  --- | --- | --- |\n\n\n')
    sb.append('\n| **Area name** | **Spec Count** | **Test Count** |\n')
    sb.append('|  --- | --- | --- |\n')
    def totalFuncSpecCount = funcSpecData.collect { it.getValue().size() }.sum()
    def totalFuncTestCount = funcTestData.collect { it.getValue() }.sum()
    sb.append("| **Total** | **$totalFuncSpecCount** | **$totalFuncTestCount** |\n")
    funcSpecData.each { area, setOfSpecs ->
        def specCount = setOfSpecs.size()
        def testCount = funcTestData.get(area)
        sb.append("| $area | $specCount | $testCount |\n")
    }
    sb.append("***")
}

/**
 * Method to prepare a markdown table for functional area's spec and test statistics
 * @param sb : string builder instance
 * @param testSuiteSpecData : Test Suite area collective spec data
 * @param testSuiteTestData : Test Suite area collective test block data
 */
private void prepareTestSuiteTable(StringBuilder sb, Map<String, Set> testSuiteSpecData, Map<String, Integer> testSuiteTestData) {
    sb.append('\n\n| Test Suite Statistics|||\n')
    sb.append('|  --- | --- | --- |\n\n\n')
    sb.append('| **Test Suite** | **Spec Count** | **Test Count** |\n')
    sb.append('|  --- | --- | --- |\n')
    def testSuiteTotalSpecCount = testSuiteSpecData.collect { it.getValue().size() }.sum()
    def totalTestSuiteTestCount = testSuiteTestData.collect { it.getValue() }.sum()
    sb.append("| **Total** | **$testSuiteTotalSpecCount** | **$totalTestSuiteTestCount** |\n")
    testSuiteSpecData.each { area, setOfSpecs ->
        def specCount = setOfSpecs.size()
        def testCount = testSuiteTestData.get(area)
        sb.append("| $area | $specCount | $testCount |\n")
    }
}

/**
 * Method to create Test Suite spec and test block maps
 * @param testSuiteTestCounterMap
 * @param tag
 * @param testSuiteSpecCounterMap
 * @param suite
 */
private void calculateTestSuiteCounters(LinkedHashMap<String, Integer> testSuiteTestCounterMap, String tag, LinkedHashMap<String, Set> testSuiteSpecCounterMap, Object suite) {
    if (!testSuiteTestCounterMap[tag] || !testSuiteSpecCounterMap[tag]) {
        testSuiteSpecCounterMap[tag] = new HashSet<String>()
        testSuiteSpecCounterMap[tag] << suite.file
        testSuiteTestCounterMap[tag] = 1
    } else {
        testSuiteTestCounterMap[tag] = testSuiteTestCounterMap[tag] + 1
        testSuiteSpecCounterMap[tag] << suite.file
    }
}

/**
 * Method to create Functional area's spec and test block maps
 * @param testSuiteTestCounterMap
 * @param tag
 * @param functionalAreaSpecCounterMap
 * @param suite
 */
private void calculateFunctionalAreaTestCounters(LinkedHashMap<String, Integer> testSuiteTestCounterMap, String tag, LinkedHashMap<String, Set> functionalAreaSpecCounterMap, Object suite) {
    if (!testSuiteTestCounterMap[tag] || !functionalAreaSpecCounterMap[tag]) {
        functionalAreaSpecCounterMap[tag] = new HashSet<String>()
        functionalAreaSpecCounterMap[tag] << suite.file
        testSuiteTestCounterMap[tag] = 1
    } else {
        testSuiteTestCounterMap[tag] = testSuiteTestCounterMap[tag] + 1
        functionalAreaSpecCounterMap[tag] << suite.file
    }
}

/**
 * This is the main method to generate coverage report called from Jenkins file
 * @param testCoverageJson : This is the json report generated by playwright
 */
String generateTestCoverageReport(testCoverageJson) {
    final List TEST_SUITE_LIST = ["sanity", "smoke", "perf", "new_flow"]
    StringBuilder sb = new StringBuilder()
    Map<String, Set> functionalAreaSpecCounterMap = [:]
    Map<String, Integer> functionalAreaTestCounterMap = [:]
    Map<String, Integer> testSuiteTestCounterMap = [:]
    Map<String, Set> testSuiteSpecCounterMap = [:]
    testCoverageJson.suites.each { suite ->
        suite.specs.each { spec ->
            spec.tags.each { String tag ->
                if (TEST_SUITE_LIST.contains(tag)) {
                    calculateTestSuiteCounters(testSuiteTestCounterMap, tag, testSuiteSpecCounterMap, suite)
                } else {
                    calculateFunctionalAreaTestCounters(functionalAreaTestCounterMap, tag, functionalAreaSpecCounterMap, suite)
                }
            }
        }
    }
    prepareFunctionalTable(sb, functionalAreaSpecCounterMap, functionalAreaTestCounterMap)
    prepareTestSuiteTable(sb, testSuiteSpecCounterMap, testSuiteTestCounterMap)
    return sb.toString()
}
