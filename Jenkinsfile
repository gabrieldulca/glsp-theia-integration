def kubernetes_config = """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: node
    image: eclipsetheia/theia-blueprint
    tty: true
    resources:
      limits:
        memory: "4Gi"
        cpu: "2"
      requests:
        memory: "4Gi"
        cpu: "2"
    command:
    - cat
    volumeMounts:
    - mountPath: "/home/jenkins"
      name: "jenkins-home"
      readOnly: false
    - mountPath: "/.yarn"
      name: "yarn-global"
      readOnly: false
  volumes:
  - name: "jenkins-home"
    emptyDir: {}
  - name: "yarn-global"
    emptyDir: {}
"""

pipeline {
    agent {
        kubernetes {
            label 'glsp-agent-pod'
            yaml kubernetes_config
        }
    }
    options {
        buildDiscarder logRotator(numToKeepStr: '15')
    }

    environment {
        YARN_CACHE_FOLDER = "${env.WORKSPACE}/yarn-cache"
        SPAWN_WRAP_SHIM_ROOT = "${env.WORKSPACE}"
    }
    
    stages {
        stage('Build') {
            steps {
                timeout(30) {
                    container('node') {
                        sh "yarn --ignore-engines --unsafe-perm"
                    }
                }
            }
        }

        stage('Codechecks (ESLint)'){
            steps {
                container('node') {
                    timeout(30){
                        sh "yarn lint -o eslint.xml -f checkstyle"                      
                    }
                }
            }
        }

        stage('Deploy (master only)') {
            when { 
                allOf {
                    branch 'master'
                    expression {  
                      /* Only trigger the deployment job if the changeset contains changes in 
                      the `packages` or `examples/workflow-theia` directory */
                      sh(returnStatus: true, script: 'git diff --name-only HEAD^ | grep --quiet "^packages\\|examples/workflow-theia"') == 0
                    }
                }
            }
            steps {
                build job: 'deploy-npm-glsp-theia-integration', wait: false
            }
        }
    }

    post {
        always {
            // Record & publish ESLint issues
            recordIssues enabledForFailure: true, publishAllIssues: true, aggregatingResults: true, 
            tools: [esLint(pattern: 'node_modules/**/*/eslint.xml')], 
            qualityGates: [[threshold: 1, type: 'TOTAL', unstable: true]]
        }
    }
}
