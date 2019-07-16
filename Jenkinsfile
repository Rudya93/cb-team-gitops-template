pipeline {
    agent {
        kubernetes {
        label 'team-automation'
        yaml """
kind: Pod
spec:
  containers:
  - name: hub
    image:  caladreas/hub
    command: ["cat"]
    tty: true
    resources:
      requests:
        memory: "50Mi"
        cpu: "150m"
      limits:
        memory: "50Mi"
        cpu: "150m"
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["cat"]
    tty: true
    securityContext:
      runAsUser: 1000
      fsGroup: 1000
    resources:
      requests:
        memory: "50Mi"
        cpu: "100m"
      limits:
        memory: "150Mi"
        cpu: "200m"
  - name: yq
    image: mikefarah/yq
    command: ['cat']
    tty: true
    resources:
      requests:
        memory: "50Mi"
        cpu: "100m"
      limits:
        memory: "50Mi"
        cpu: "100m"
  - name: jq
    image: colstrom/jq
    command: ['cat']
    tty: true
    resources:
      requests:
        memory: "50Mi"
        cpu: "100m"
      limits:
        memory: "50Mi"
        cpu: "100m"
        
"""
//   securityContext:
//     runAsUser: 1000
//     fsGroup: 1000
        }
    }
    libraries {
        lib('github.com/joostvdg/jpl-core')
        //lib('github.com/joostvdg/jpl-gitops-kubernetes')
    }
    options {
        disableConcurrentBuilds()
        checkoutToSubdirectory 'templates' // we need to do two checkouts
        buildDiscarder logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '5', numToKeepStr: '5')
    }
    environment {
        envGitInfo          = ''
        RESET_NAMESPACE     = 'jx-production'
        TEAM_BASE_NAME      = ''
        NAMESPACE_TO_CREATE = ''
        DISPLAY_NAME        = ''
        TEAM_RECIPE         = ''
        ICON                = ''
        ICON_COLOR_CODE     = ''
        ADMINS_ROLE         = ''
        MEMBERS_ROLE        = ''
        GUESTS_ROLE         = ''
        RECORD_LOC          = ''
    }
    stages {
        stage('Team Details') {
            input {
                message "Please enter the team details"
                ok "Looks good, proceed"
                parameters {
                    string(name: 'Name', defaultValue: 'hex', description: 'Please specify a team name')
                    string(name: 'DisplayName', defaultValue: 'Hex', description: 'Please specify a team display name')
                    choice choices: ['joostvdg', 'basic', 'java-web'], description: 'Please select a Team Recipe', name: 'TeamRecipe'
                    choice choices: ['anchor', 'bear', 'bowler-hat', 'briefcase', 'bug', 'calculator', 'calculatorcart', 'clock', 'cloud', 'cloudbees', 'connect', 'dollar-bill', 'dollar-symbol', 'file', 'flag', 'flower-carnation', 'flower-daisy', 'help', 'hexagon', 'high-heels', 'jenkins', 'key', 'marker', 'monocle', 'mustache', 'office', 'panther', 'paw-print', 'teacup', 'tiger', 'truck'], description: 'Please select an Icon', name: 'Icon'
                    string(name: 'IconColorCode', defaultValue: '#CCCCCC', description: 'Please specify a valid html hexcode for the color (https://htmlcolorcodes.com/)')
                    string(name: 'Admins', defaultValue: 'Catmins', description: 'Please specify a groupid or userid for the TEAM_ADMIN role')
                    string(name: 'Members', defaultValue: 'Pirates', description: 'Please specify a groupid or userid for the TEAM_MEMBER role')
                    string(name: 'Guests', defaultValue: 'Continental', description: 'Please specify a groupid or userid for the TEAM_GUEST role')
                }
            }
            steps {
                println "Name=${Name}"
                println "DisplayName=${DisplayName}"
                println "TeamRecipe=${TeamRecipe}"
                println "Icon=${Icon}"
                println "IconColorCode=${IconColorCode}"
                println "Admins=${Admins}"
                println "Members=${Members}"
                println "Guests=${Guests}"
                script {
                    TEAM_BASE_NAME      = "${Name}"
                    NAMESPACE_TO_CREATE = "cb-teams-${Name}"
                    DISPLAY_NAME        = "${DisplayName}"
                    TEAM_RECIPE         = "${TeamRecipe}"
                    ICON                = "${Icon}"
                    ICON_COLOR_CODE     = "${IconColorCode}"
                    ADMINS_ROLE         = "${Admins}"
                    MEMBERS_ROLE        = "${Members}"
                    GUESTS_ROLE         = "${Guests}"
                    RECORD_LOC          = "templates/teams/${Name}"
                    sh "mkdir -p ${RECORD_LOC}"
                }
            }
        }
        stage('Create Team Config') {
            environment {
                BASE        = 'templates/namespace-creation/kustomize'
                NAMESPACE   = "${NAMESPACE_TO_CREATE}"
                RECORD_LOC  = "templates/teams/${TEAM_BASE_NAME}"
            }
            parallel {
                stage('Namespace') {
                    steps {
                        container('yq') {
                            sh 'yq w -i ${BASE}/base/role-binding.yaml subjects[0].namespace ${NAMESPACE}'
                            sh 'yq w -i ${BASE}/base/namespace.yaml metadata.name ${NAMESPACE}'
                            sh 'yq w -i ${BASE}/team-example/kustomization.yaml namespace ${NAMESPACE}'
                        }
                        container('kubectl') {
                            sh '''
                                kubectl kustomize ${BASE}/team-example > ${RECORD_LOC}/team.yaml
                                cat ${RECORD_LOC}/team.yaml
                            '''
                        }
                    }
                }
                stage('Team Master JSON') {
                    steps {
                        container('jq') {
                            sh """jq \
                            '.data.name = "${TEAM_BASE_NAME}" |\
                            .data.displayName = "${DISPLAY_NAME}" |\
                            .data.provisioningRecipe = "${TEAM_RECIPE}" |\
                            .data.icon.name = "${ICON}" |\
                            .data.icon.color = "${ICON_COLOR_CODE}" |\
                            .data.members[0].id = "${ADMINS_ROLE}" |\
                            .data.members[1].id = "${MEMBERS_ROLE}" |\
                            .data.members[2].id = "${GUESTS_ROLE}"'\
                            templates/team-master-template/simple.json > ${RECORD_LOC}/team.json
                            """
                        }
                        sh 'cat ${RECORD_LOC}/team.json'
                    }
                }
            }
        }
        // stage('Stash') {
        //     steps {
        //         stash includes: "teams/${TEAM_BASE_NAME}/team.*", name: 'new_team'
        //     }
        // }
        stage('Create PR') {
            when { branch 'master'}
            environment {
                RECORD_LOC      = "templates/teams/${TEAM_BASE_NAME}"
                PR_CHANGE_NAME  = "add_team_${TEAM_BASE_NAME}"
            }
            steps {
                container('hub') {
                    dir('cb-team-gitops') {
                        script {
                            envGitInfo = git 'https://github.com/joostvdg/cb-team-gitops.git'
                        }
                        sh 'git checkout -b ${PR_CHANGE_NAME}'
                        sh 'copy -R ../${RECORD_LOC} ./teams'
                        sh 'ls -lath'
                        sh 'ls -lath teams/'

                        gitRemoteConfigByUrl(envGitInfo.GIT_URL, 'githubtoken_token') // must be a API Token ONLY -> secret text
                        sh '''
                        git config --global user.email "jenkins@jenkins.io"
                        git config --global user.name "Jenkins"
                        git add ${RECORD_LOC}
                        git status
                        git commit -m "add team ${TEAM_BASE_NAME}"
                        git push origin ${PR_CHANGE_NAME}
                        '''


                        // has to be indented like that, else the indents will be in the pr description
                        writeFile encoding: 'UTF-8', file: 'pr-info.md', text: """Add ${TEAM_BASE_NAME}
\n
This pr is automatically generated via CloudBees.\\n
\n
The job: ${env.JOB_URL}
                    """

                        // TODO: unfortunately, environment {}'s credentials have fixed environment variable names
                        // TODO: in this case, they need to be EXACTLY GITHUB_PASSWORD and GITHUB_USER
                        script {
                            withCredentials([usernamePassword(credentialsId: 'githubtoken', passwordVariable: 'GITHUB_PASSWORD', usernameVariable: 'GITHUB_USER')]) {
                                sh """
                                set +x
                                hub pull-request --force -F pr-info.md -l '${TEAM_BASE_NAME}' --no-edit
                                """
                            }
                        }
                    }
                }
            }
        }
    }
}
