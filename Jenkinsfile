// ── MUTABLE RESULT TRACKING ───────────────────────────────────────────────────
// Declared outside the pipeline block so they are true Groovy script-level
// variables. These are readable AND writable from both `script {}` steps and
// `when { expression {} }` blocks — unlike `environment {}` variables which
// are frozen after the environment block evaluates.
def buildResult           = 'NOT_RUN'
def unitTestResult        = 'NOT_RUN'
def integrationTestResult = 'NOT_RUN'
def stagingDeployResult   = 'NOT_RUN'
def prodDeployResult      = 'NOT_RUN'
def smokeTestResult       = 'NOT_RUN'

pipeline {
    agent any

    parameters {
        // ── Downstream job names ──────────────────────────────────────────────
        string(name: 'BUILD_JOB',            defaultValue: 'build-job',            description: 'Compile / package job')
        string(name: 'UNIT_TEST_JOB',        defaultValue: 'unit-test-job',        description: 'Unit-test job (runs in parallel with integration tests)')
        string(name: 'INTEGRATION_TEST_JOB', defaultValue: 'integration-test-job', description: 'Integration-test job (runs in parallel with unit tests)')
        string(name: 'STAGING_DEPLOY_JOB',   defaultValue: 'staging-deploy-job',   description: 'Deploy to staging environment')
        string(name: 'PROD_DEPLOY_JOB',      defaultValue: 'prod-deploy-job',      description: 'Deploy to production environment')
        string(name: 'ROLLBACK_JOB',         defaultValue: 'rollback-job',         description: 'Rollback job triggered on deploy failure')
        string(name: 'NOTIFY_JOB',           defaultValue: 'notify-job',           description: 'Notification job (always runs)')
        string(name: 'SMOKE_TEST_JOB',       defaultValue: 'smoke-test-job',       description: 'Post-deploy smoke tests')

        // ── Behavioural flags ─────────────────────────────────────────────────
        booleanParam(name: 'RUN_TESTS',          defaultValue: true,  description: 'Run unit + integration tests after build')
        booleanParam(name: 'RUN_STAGING_DEPLOY', defaultValue: true,  description: 'Deploy to staging when tests pass')
        booleanParam(name: 'RUN_PROD_DEPLOY',    defaultValue: false, description: 'Deploy to production (requires manual gate)')
        booleanParam(name: 'RUN_SMOKE_TESTS',    defaultValue: true,  description: 'Run smoke tests after each deploy')
        booleanParam(name: 'ENABLE_ROLLBACK',    defaultValue: true,  description: 'Auto-trigger rollback on deploy failure')

        // ── Context ───────────────────────────────────────────────────────────
        string(name: 'TARGET_BRANCH',  defaultValue: 'main',    description: 'Branch allowed to deploy to production')
        string(name: 'ARTIFACT_ID',    defaultValue: '',        description: 'Artifact identifier passed downstream (auto-set if blank)')
        choice(name: 'DEPLOY_REGION',  choices: ['us-east-1', 'eu-west-1', 'ap-southeast-1'], description: 'Target cloud region')
    }

    environment {
        // Only static values here — derived artifact ID (read-only after set)
        EFFECTIVE_ARTIFACT_ID = "${params.ARTIFACT_ID?.trim() ? params.ARTIFACT_ID : env.BUILD_NUMBER}"
    }

    stages {

        // ── 1. BUILD ──────────────────────────────────────────────────────────
        stage('Build') {
            steps {
                script {
                    echo "==> Triggering build job: ${params.BUILD_JOB}"
                    def run = build job: params.BUILD_JOB,
                                   propagate: false,
                                   wait: true,
                                   parameters: [
                                       string(name: 'ARTIFACT_ID', value: env.EFFECTIVE_ARTIFACT_ID)
                                   ]

                    buildResult = run.result ?: 'SUCCESS'
                    echo "Build result: ${buildResult}"

                    if (buildResult != 'SUCCESS') {
                        currentBuild.result = 'FAILURE'
                        error("Build job failed — aborting pipeline.")
                    }
                }
            }
        }

        // ── 2. PARALLEL TESTS ─────────────────────────────────────────────────
        stage('Tests') {
            when {
                expression { return params.RUN_TESTS && buildResult == 'SUCCESS' }
            }
            parallel {
                stage('Unit Tests') {
                    steps {
                        script {
                            echo "==> Triggering unit-test job: ${params.UNIT_TEST_JOB}"
                            def run = build job: params.UNIT_TEST_JOB,
                                           propagate: false,
                                           wait: true,
                                           parameters: [
                                               string(name: 'ARTIFACT_ID', value: env.EFFECTIVE_ARTIFACT_ID)
                                           ]

                            unitTestResult = run.result ?: 'SUCCESS'
                            echo "Unit-test result: ${unitTestResult}"
                        }
                    }
                }

                stage('Integration Tests') {
                    steps {
                        script {
                            echo "==> Triggering integration-test job: ${params.INTEGRATION_TEST_JOB}"
                            def run = build job: params.INTEGRATION_TEST_JOB,
                                           propagate: false,
                                           wait: true,
                                           parameters: [
                                               string(name: 'ARTIFACT_ID', value: env.EFFECTIVE_ARTIFACT_ID)
                                           ]

                            integrationTestResult = run.result ?: 'SUCCESS'
                            echo "Integration-test result: ${integrationTestResult}"
                        }
                    }
                }
            }

            post {
                always {
                    script {
                        if (unitTestResult != 'SUCCESS' || integrationTestResult != 'SUCCESS') {
                            currentBuild.result = 'FAILURE'
                            echo "One or more test jobs failed. Unit=${unitTestResult}, Integration=${integrationTestResult}"
                        }
                    }
                }
            }
        }

        // ── 3. STAGING DEPLOY ─────────────────────────────────────────────────
        stage('Deploy to Staging') {
            when {
                expression {
                    return params.RUN_STAGING_DEPLOY &&
                           buildResult == 'SUCCESS' &&
                           unitTestResult in ['SUCCESS', 'NOT_RUN'] &&
                           integrationTestResult in ['SUCCESS', 'NOT_RUN']
                }
            }
            steps {
                script {
                    echo "==> Triggering staging deploy: ${params.STAGING_DEPLOY_JOB}"
                    def run = build job: params.STAGING_DEPLOY_JOB,
                                   propagate: false,
                                   wait: true,
                                   parameters: [
                                       string(name: 'ARTIFACT_ID',   value: env.EFFECTIVE_ARTIFACT_ID),
                                       string(name: 'DEPLOY_REGION', value: params.DEPLOY_REGION),
                                       string(name: 'ENVIRONMENT',   value: 'staging')
                                   ]

                    stagingDeployResult = run.result ?: 'SUCCESS'
                    echo "Staging deploy result: ${stagingDeployResult}"

                    if (stagingDeployResult != 'SUCCESS' && params.ENABLE_ROLLBACK) {
                        triggerRollback('staging')
                    }
                }
            }
        }

        // ── 4. STAGING SMOKE TESTS ────────────────────────────────────────────
        stage('Smoke Tests (Staging)') {
            when {
                expression {
                    return params.RUN_SMOKE_TESTS &&
                           stagingDeployResult == 'SUCCESS'
                }
            }
            steps {
                script {
                    echo "==> Triggering smoke tests against staging: ${params.SMOKE_TEST_JOB}"
                    def run = build job: params.SMOKE_TEST_JOB,
                                   propagate: false,
                                   wait: true,
                                   parameters: [
                                       string(name: 'ARTIFACT_ID', value: env.EFFECTIVE_ARTIFACT_ID),
                                       string(name: 'ENVIRONMENT', value: 'staging')
                                   ]

                    smokeTestResult = run.result ?: 'SUCCESS'
                    echo "Smoke test result (staging): ${smokeTestResult}"

                    if (smokeTestResult != 'SUCCESS') {
                        currentBuild.result = 'UNSTABLE'
                        echo "Smoke tests failed on staging — production deploy will be blocked."
                    }
                }
            }
        }

        // ── 5. MANUAL GATE ────────────────────────────────────────────────────
        stage('Approve Production Deploy') {
            when {
                expression {
                    return params.RUN_PROD_DEPLOY &&
                           env.BRANCH_NAME == params.TARGET_BRANCH &&
                           stagingDeployResult == 'SUCCESS' &&
                           smokeTestResult == 'SUCCESS'
                }
            }
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: "Deploy artifact '${env.EFFECTIVE_ARTIFACT_ID}' to PRODUCTION (${params.DEPLOY_REGION})?",
                          ok: 'Deploy',
                          submitter: 'release-team'
                }
            }
        }

        // ── 6. PRODUCTION DEPLOY ──────────────────────────────────────────────
        stage('Deploy to Production') {
            when {
                expression {
                    return params.RUN_PROD_DEPLOY &&
                           env.BRANCH_NAME == params.TARGET_BRANCH &&
                           stagingDeployResult == 'SUCCESS' &&
                           smokeTestResult == 'SUCCESS' &&
                           currentBuild.result in [null, 'SUCCESS']
                }
            }
            steps {
                script {
                    echo "==> Triggering production deploy: ${params.PROD_DEPLOY_JOB}"
                    def run = build job: params.PROD_DEPLOY_JOB,
                                   propagate: false,
                                   wait: true,
                                   parameters: [
                                       string(name: 'ARTIFACT_ID',   value: env.EFFECTIVE_ARTIFACT_ID),
                                       string(name: 'DEPLOY_REGION', value: params.DEPLOY_REGION),
                                       string(name: 'ENVIRONMENT',   value: 'production')
                                   ]

                    prodDeployResult = run.result ?: 'SUCCESS'
                    echo "Production deploy result: ${prodDeployResult}"

                    if (prodDeployResult != 'SUCCESS') {
                        currentBuild.result = 'FAILURE'
                        if (params.ENABLE_ROLLBACK) {
                            triggerRollback('production')
                        }
                    }
                }
            }
        }

        // ── 7. PRODUCTION SMOKE TESTS ─────────────────────────────────────────
        stage('Smoke Tests (Production)') {
            when {
                expression {
                    return params.RUN_SMOKE_TESTS &&
                           prodDeployResult == 'SUCCESS'
                }
            }
            steps {
                script {
                    echo "==> Triggering smoke tests against production: ${params.SMOKE_TEST_JOB}"
                    def run = build job: params.SMOKE_TEST_JOB,
                                   propagate: false,
                                   wait: true,
                                   parameters: [
                                       string(name: 'ARTIFACT_ID', value: env.EFFECTIVE_ARTIFACT_ID),
                                       string(name: 'ENVIRONMENT', value: 'production')
                                   ]

                    smokeTestResult = run.result ?: 'SUCCESS'
                    echo "Smoke test result (production): ${smokeTestResult}"

                    if (smokeTestResult != 'SUCCESS') {
                        currentBuild.result = 'FAILURE'
                        if (params.ENABLE_ROLLBACK) {
                            triggerRollback('production')
                        }
                    }
                }
            }
        }
    }

    // ── POST ──────────────────────────────────────────────────────────────────
    post {
        always {
            script {
                if (params.NOTIFY_JOB?.trim()) {
                    echo "==> Triggering notification job: ${params.NOTIFY_JOB}"
                    build job: params.NOTIFY_JOB,
                          propagate: false,
                          wait: false,
                          parameters: [
                              string(name: 'PIPELINE_NAME',           value: env.JOB_NAME),
                              string(name: 'PIPELINE_RESULT',         value: currentBuild.currentResult),
                              string(name: 'ARTIFACT_ID',             value: env.EFFECTIVE_ARTIFACT_ID),
                              string(name: 'BUILD_RESULT',            value: buildResult),
                              string(name: 'UNIT_TEST_RESULT',        value: unitTestResult),
                              string(name: 'INTEGRATION_TEST_RESULT', value: integrationTestResult),
                              string(name: 'STAGING_DEPLOY_RESULT',   value: stagingDeployResult),
                              string(name: 'PROD_DEPLOY_RESULT',      value: prodDeployResult),
                              string(name: 'SMOKE_TEST_RESULT',       value: smokeTestResult),
                              string(name: 'BUILD_URL',               value: env.BUILD_URL)
                          ]
                }
            }
        }

        success {
            echo "Pipeline completed successfully. Artifact: ${env.EFFECTIVE_ARTIFACT_ID}"
        }

        failure {
            echo "Pipeline failed. Check individual job results above for details."
        }

        unstable {
            echo "Pipeline is unstable (smoke tests failed on staging)."
        }
    }
}

// ── HELPER FUNCTIONS ──────────────────────────────────────────────────────────

def triggerRollback(String environment) {
    echo "==> Deploy failed on ${environment}. Triggering rollback: ${params.ROLLBACK_JOB}"
    build job: params.ROLLBACK_JOB,
          propagate: false,
          wait: false,
          parameters: [
              string(name: 'ARTIFACT_ID',   value: env.EFFECTIVE_ARTIFACT_ID),
              string(name: 'ENVIRONMENT',   value: environment),
              string(name: 'DEPLOY_REGION', value: params.DEPLOY_REGION)
          ]
}
