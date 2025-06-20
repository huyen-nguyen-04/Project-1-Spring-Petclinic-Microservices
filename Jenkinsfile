pipeline {
    agent any

    environment {
        REPOSITORY = 'git@github.com:huyen-nguyen-04/Project-1-Spring-Petclinic-Microservices.git'
    }

    stages {
        stage('Detect Changed Service') {
            steps {
                script {
                    LATEST_COMMIT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    echo "Latest Commit Hash: ${LATEST_COMMIT}"

                    CHANGED_SERVICE = sh(script: """
                        git diff-tree -m --no-commit-id --name-only -r ${LATEST_COMMIT} | \
                        grep '^spring-petclinic-.*/' | \
                        head -n 1
                    """, returnStdout: true).trim().split('/')[0]
                    echo "Changed Service: ${CHANGED_SERVICE}"
                }
            }
        }

        stage('Test') {
            when {
                expression {
                    return CHANGED_SERVICE != ''
                }
            }

            steps {
                script {
                    echo "Testing ${CHANGED_SERVICE} ..."
                    sh "./mvnw clean verify -f ${CHANGED_SERVICE}/pom.xml"
                    junit "${CHANGED_SERVICE}/target/surefire-reports/*.xml"
                    jacoco (
                        execPattern: "${CHANGED_SERVICE}/target/jacoco.exec",
                        classPattern: "${CHANGED_SERVICE}/target/classes",
                        sourcePattern: "${CHANGED_SERVICE}/src/main/java",
                        exclusionPattern: "${CHANGED_SERVICE}/target/test-classes"
                    )
                    def coveragePercentage = getCoveragePercentage("${CHANGED_SERVICE}/target/site/jacoco/jacoco.csv")
                    if (coveragePercentage < 0.7) {
                        echo "Code coverage for ${CHANGED_SERVICE} is below 70: ${coveragePercentage * 100}"
                        error "Code coverage for ${CHANGED_SERVICE} is below 70: ${coveragePercentage * 100}"
                    } else {
                        echo "Code coverage for ${CHANGED_SERVICE} is ${coveragePercentage * 100}"
                    }
                    echo "${CHANGED_SERVICE} test completed."
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    echo "Building ${CHANGED_SERVICE} ..."
                    sh "./mvnw clean install -f ${CHANGED_SERVICE}/pom.xml -DskipTests"
                    echo "${CHANGED_SERVICE} build completed."
                }
            }
        }
    }

    post {
        success {
            setBuildStatus("Build Successful", "SUCCESS")
        }

        failure {
            setBuildStatus("Build Failed", "FAILURE")
        }
    }


}

void setBuildStatus(String message, String state) {
    step([
        $class: "GitHubCommitStatusSetter",
        reposSource: [$class: "ManuallyEnteredRepositorySource", url: REPOSITORY],
        contextSource: [$class: "ManuallyEnteredCommitContextSource", context: "ci/jenkins/build-status"],
        errorHandlers: [[$class: "ChangingBuildStatusErrorHandler", result: "UNSTABLE"]],
        statusResultSource: [$class: "ConditionalStatusResultSource", results: [[$class: "AnyBuildResult", message: message, state: state]]]
    ]);
}

double getCoveragePercentage(String filepath) {
    def fileContents = readFile(filepath)
    def totalMissed = 0
    def totalCovered = 0

    fileContents.split('\n').eachWithIndex { line, index ->
        if (index == 0) return 

        def columns = line.split(",")
        totalMissed += columns[3].toInteger() + columns[5].toInteger() + columns[7].toInteger() + columns[9].toInteger() + columns[11].toInteger()
        totalCovered += columns[4].toInteger() + columns[6].toInteger() + columns[8].toInteger() + columns[10].toInteger() + columns[12].toInteger()
    }

    return (totalCovered / (totalCovered + totalMissed))
}