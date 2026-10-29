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
        // Rolled-up results — initialised to NOT_RUN so conditions read cleanly
        BUILD_RESULT           = 'NOT_RUN'
        UNIT_TEST_RESULT       = 'NOT_RUN'
        INTEGRATION_TEST_RESULT= 'NOT_RUN'
        STAGING_DEPLOY_RESULT  = 'NOT_RUN'
        PROD_DEPLOY_RESULT     = 'NOT_RUN'
        SMOKE_TEST_RESULT      = 'NOT_RUN'

        // Derived artifact ID — prefer the explicit param, fall back to build number
        EFFECTIVE_ARTIFACT_ID  = "${params.ARTIFACT_ID?.trim() ? params.ARTIFACT_ID : env.BUILD_NUMBER}"
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

                    env.BUILD_RESULT = run.result ?: 'SUCCESS'
                    echo "Build result: ${env.BUILD_RESULT}"

                    if (env.BUILD_RESULT != 'SUCCESS') {
                        currentBuild.result = 'FAILURE'
                        error("Build job failed — aborting pipeline.")
                    }
                }
            }
        }

        // ── 2. PARALLEL TESTS ─────────────────────────────────────────────────
        // Runs unit and integration tests side-by-side.
        // Both must pass before deployment is considered.
        stage('Tests') {
            when {
                expression { return params.RUN_TESTS && env.BUILD_RESULT == 'SUCCESS' }
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

                            env.UNIT_TEST_RESULT = run.result ?: 'SUCCESS'
                            echo "Unit-test result: ${env.UNIT_TEST_RESULT}"
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

                            env.INTEGRATION_TEST_RESULT = run.result ?: 'SUCCESS'
                            echo "Integration-test result: ${env.INTEGRATION_TEST_RESULT}"
                        }
                    }
                }
            }

            // Evaluate combined test outcome after both parallel legs finish
            post {
                always {
                    script {
                        def unitOk        = env.UNIT_TEST_RESULT        == 'SUCCESS'
                        def integrationOk = env.INTEGRATION_TEST_RESULT == 'SUCCESS'

                        if (!unitOk || !integrationOk) {
                            currentBuild.result = 'FAILURE'
                            echo "One or more test jobs failed. Unit=${env.UNIT_TEST_RESULT}, Integration=${env.INTEGRATION_TEST_RESULT}"
                        }
                    }
                }
            }
        }

        // ── 3. STAGING DEPLOY ─────────────────────────────────────────────────
        // Deploys to staging when build + both tests passed.
        stage('Deploy to Staging') {
            when {
                expression {
                    return params.RUN_STAGING_DEPLOY &&
                           env.BUILD_RESULT == 'SUCCESS' &&
                           allTestsPassed()
                }
            }
            steps {
                script {
                    echo "==> Triggering staging deploy: ${params.STAGING_DEPLOY_JOB}"
                    def run = build job: params.STAGING_DEPLOY_JOB,
                                   propagate: false,
                                   wait: true,
                                   parameters: [
                                       string(name: 'ARTIFACT_ID',    value: env.EFFECTIVE_ARTIFACT_ID),
                                       string(name: 'DEPLOY_REGION',  value: params.DEPLOY_REGION),
                                       string(name: 'ENVIRONMENT',    value: 'staging')
                                   ]

                    env.STAGING_DEPLOY_RESULT = run.result ?: 'SUCCESS'
                    echo "Staging deploy result: ${env.STAGING_DEPLOY_RESULT}"

                    if (env.STAGING_DEPLOY_RESULT != 'SUCCESS' && params.ENABLE_ROLLBACK) {
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
                           env.STAGING_DEPLOY_RESULT == 'SUCCESS'
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

                    env.SMOKE_TEST_RESULT = run.result ?: 'SUCCESS'
                    echo "Smoke test result (staging): ${env.SMOKE_TEST_RESULT}"

                    if (env.SMOKE_TEST_RESULT != 'SUCCESS') {
                        currentBuild.result = 'UNSTABLE'
                        echo "Smoke tests failed on staging — production deploy will be blocked."
                    }
                }
            }
        }

        // ── 5. MANUAL GATE ────────────────────────────────────────────────────
        // Human approval required before touching production.
        // Only reached when: prod deploy is enabled, branch is correct,
        // staging deployed and smoke tests passed.
        stage('Approve Production Deploy') {
            when {
                expression {
                    return params.RUN_PROD_DEPLOY &&
                           env.BRANCH_NAME == params.TARGET_BRANCH &&
                           env.STAGING_DEPLOY_RESULT == 'SUCCESS' &&
                           env.SMOKE_TEST_RESULT == 'SUCCESS'
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
                // Re-check all guards here — the input step above may have been
                // approved but conditions could have changed (e.g. concurrent run).
                expression {
                    return params.RUN_PROD_DEPLOY &&
                           env.BRANCH_NAME == params.TARGET_BRANCH &&
                           env.STAGING_DEPLOY_RESULT == 'SUCCESS' &&
                           env.SMOKE_TEST_RESULT == 'SUCCESS' &&
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

                    env.PROD_DEPLOY_RESULT = run.result ?: 'SUCCESS'
                    echo "Production deploy result: ${env.PROD_DEPLOY_RESULT}"

                    if (env.PROD_DEPLOY_RESULT != 'SUCCESS') {
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
                           env.PROD_DEPLOY_RESULT == 'SUCCESS'
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

                    // Reuse the same env var — last write wins (production)
                    env.SMOKE_TEST_RESULT = run.result ?: 'SUCCESS'
                    echo "Smoke test result (production): ${env.SMOKE_TEST_RESULT}"

                    if (env.SMOKE_TEST_RESULT != 'SUCCESS') {
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
                // Notify regardless of outcome so teams always get a status update.
                if (params.NOTIFY_JOB?.trim()) {
                    echo "==> Triggering notification job: ${params.NOTIFY_JOB}"
                    build job: params.NOTIFY_JOB,
                          propagate: false,
                          wait: false,
                          parameters: [
                              string(name: 'PIPELINE_NAME',              value: env.JOB_NAME),
                              string(name: 'PIPELINE_RESULT',            value: currentBuild.currentResult),
                              string(name: 'ARTIFACT_ID',                value: env.EFFECTIVE_ARTIFACT_ID),
                              string(name: 'BUILD_RESULT',               value: env.BUILD_RESULT),
                              string(name: 'UNIT_TEST_RESULT',           value: env.UNIT_TEST_RESULT),
                              string(name: 'INTEGRATION_TEST_RESULT',    value: env.INTEGRATION_TEST_RESULT),
                              string(name: 'STAGING_DEPLOY_RESULT',      value: env.STAGING_DEPLOY_RESULT),
                              string(name: 'PROD_DEPLOY_RESULT',         value: env.PROD_DEPLOY_RESULT),
                              string(name: 'SMOKE_TEST_RESULT',          value: env.SMOKE_TEST_RESULT),
                              string(name: 'BUILD_URL',                  value: env.BUILD_URL)
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

/** Returns true only when all executed test jobs passed (or were skipped). */
def allTestsPassed() {
    def unitOk        = env.UNIT_TEST_RESULT        in ['SUCCESS', 'NOT_RUN']
    def integrationOk = env.INTEGRATION_TEST_RESULT in ['SUCCESS', 'NOT_RUN']
    return unitOk && integrationOk
}

/**
 * Fire-and-forget rollback trigger.
 * @param environment  'staging' or 'production'
 */
def triggerRollback(String environment) {
    echo "==> Deploy failed on ${environment}. Triggering rollback: ${params.ROLLBACK_JOB}"
    build job: params.ROLLBACK_JOB,
          propagate: false,
          wait: false,
          parameters: [
              string(name: 'ARTIFACT_ID',  value: env.EFFECTIVE_ARTIFACT_ID),
              string(name: 'ENVIRONMENT',  value: environment),
              string(name: 'DEPLOY_REGION', value: params.DEPLOY_REGION)
          ]
}
