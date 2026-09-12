pipeline {
    agent any

    options {
        buildDiscarder(logRotator(daysToKeepStr: '7', numToKeepStr: '10'))
        overrideIndexTriggers(true)
    }

    environment {
        GITHUB_TOKEN_ID = 'ghcr-auth'
        GIT_CREDENTIALS_ID = '0ba8a1cb-47ca-4104-9c3f-d66d67f99139'
    }

    triggers {
        pollSCM('H * * * *')
    }

    stages {
        stage('Initialize GitHub Status') {
            when {
                expression { env.CHANGE_ID != null }
            }
            steps {
                script {
                    echo "Pull Request detected: #${env.CHANGE_ID}"
                    updateGithubStatus('pending', 'Build started', 'Jenkins Build')
                }
            }
        }

        stage('Validate Configuration') {
            steps {
                echo "Validating docker-compose.yml configuration..."
                sh 'docker compose config'
            }
        }

        stage('Deploy Services') {
            when {
                anyOf {
                    branch 'master'
                    branch 'main'
                }
            }
            steps {
                echo "Deploying centralized Watchtower stack on host..."
                sh 'docker network create consul_consul || true'
                sh 'docker compose up -d --remove-orphans'
            }
        }

        stage('GitHub Approve & Merge') {
            when {
                expression { env.CHANGE_ID != null }
            }
            steps {
                script {
                    updateGithubStatus('success', 'Build successful', 'Jenkins Build')
                    approveGithubPullRequest(env.CHANGE_ID)
                    mergeGithubPullRequest(env.CHANGE_ID)
                }
            }
        }
    }

    post {
        success {
            echo "Build and deployment finished successfully!"
        }
        failure {
            script {
                if (env.CHANGE_ID) {
                    updateGithubStatus('failure', 'Build failed', 'Jenkins Build')
                }
            }
            echo "Build failed!"
        }
    }
}

def updateGithubStatus(state, description, context) {
    def gitUrl = env.GIT_URL ?: "git@github.com:matheussouza88/watchtower.git"
    def repoPath = gitUrl.replace('git@github.com:', '').replace('https://github.com/', '').replace('.git', '')
    def sha = env.GIT_COMMIT ?: sh(script: 'git rev-parse HEAD', returnStdout: true).trim()

    withCredentials([usernamePassword(credentialsId: env.GITHUB_TOKEN_ID, usernameVariable: 'GH_USER', passwordVariable: 'GH_PAT')]) {
        sh """
            curl -f -s -X POST \
                -H "Authorization: Bearer \$GH_PAT" \
                -H "Accept: application/vnd.github.v3+json" \
                https://api.github.com/repos/${repoPath}/statuses/${sha} \
                -d '{"state": "${state.toLowerCase()}", "target_url": "${env.BUILD_URL}", "description": "${description}", "context": "${context}"}'
        """
    }
}

def approveGithubPullRequest(prId) {
    def gitUrl = env.GIT_URL ?: "git@github.com:matheussouza88/watchtower.git"
    def repoPath = gitUrl.replace('git@github.com:', '').replace('https://github.com/', '').replace('.git', '')

    echo "Approving Pull Request #${prId} for ${repoPath}..."

    withCredentials([usernamePassword(credentialsId: env.GITHUB_TOKEN_ID, usernameVariable: 'GH_USER', passwordVariable: 'GH_PAT')]) {
        sh """
            curl -f -s -X POST \
                -H "Authorization: Bearer \$GH_PAT" \
                -H "Accept: application/vnd.github.v3+json" \
                https://api.github.com/repos/${repoPath}/pulls/${prId}/reviews \
                -d '{"event": "APPROVE", "body": "Jenkins Build Successful. Automatically approving PR."}' || true
        """
    }
}

def mergeGithubPullRequest(prId) {
    def gitUrl = env.GIT_URL ?: "git@github.com:matheussouza88/watchtower.git"
    def repoPath = gitUrl.replace('git@github.com:', '').replace('https://github.com/', '').replace('.git', '')
    def branchName = env.CHANGE_BRANCH

    echo "Merging Pull Request #${prId} (branch: ${branchName}) for ${repoPath}..."

    withCredentials([usernamePassword(credentialsId: env.GITHUB_TOKEN_ID, usernameVariable: 'GH_USER', passwordVariable: 'GH_PAT')]) {
        def response = sh(
            script: """
                curl -f -s -w "%{http_code}" -X PUT \
                    -H "Authorization: Bearer \$GH_PAT" \
                    -H "Accept: application/vnd.github.v3+json" \
                    https://api.github.com/repos/${repoPath}/pulls/${prId}/merge \
                    -d '{"merge_method": "squash", "commit_title": "Auto-merge PR #${prId} after successful Jenkins build"}'
            """,
            returnStdout: true
        ).trim()

        def httpCode = response[-3..-1]
        def responseBody = response[0..-4]

        if (httpCode == "200" || httpCode == "201") {
            echo "Merge successful. Deleting branch ${branchName}..."
            sh """
                curl -f -s -X DELETE \
                    -H "Authorization: Bearer \$GH_PAT" \
                    -H "Accept: application/vnd.github.v3+json" \
                    https://api.github.com/repos/${repoPath}/git/refs/heads/${branchName}
            """
        } else {
            error "Merge failed with status ${httpCode}: ${responseBody}. Branch was NOT deleted."
        }
    }
}
