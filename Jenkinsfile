pipeline {
    agent any

    environment {
        TARGET_GROUP_ARN = 'arn:aws:elasticloadbalancing:il-central-1:314525640319:targetgroup/tg-umat-haash/d6712b9674576b51'
        ANSIBLE_PLAYBOOK = 'playbook.yml'
        INVENTORY_FILE = 'inventory.ini'
        AWS_DEFAULT_REGION = 'il-central-1'
    }

    stages {
        stage('Fetch EC2 Instances from Target Group') {
            steps {
                script {
                    def targetJson = sh(script: "aws elbv2 describe-target-health --target-group-arn $TARGET_GROUP_ARN", returnStdout: true).trim()
                    def instanceIds = new groovy.json.JsonSlurperClassic().parseText(targetJson).targetHealthDescriptions.collect { it.target.id }
                    env.INSTANCE_IDS = instanceIds.join(' ')
                }
            }
        }

        stage('Deregister, Run Ansible, and Reregister') {
            steps {
                sshagent(['aws']) {
                    script {
                        def failed = false
                        try {
                            for (id in env.INSTANCE_IDS.split()) {
                                sh "aws elbv2 deregister-targets --target-group-arn $TARGET_GROUP_ARN --targets Id=${id}"
                            }

                            sh "ansible-playbook -i ${env.INVENTORY_FILE} ${env.ANSIBLE_PLAYBOOK}"
                        } catch (err) {
                            failed = true
                        } finally {
                            for (id in env.INSTANCE_IDS.split()) {
                                sh "aws elbv2 register-targets --target-group-arn $TARGET_GROUP_ARN --targets Id=${id}"
                            }
                            if (failed) {
                                error("Ansible or deregistration failed. Instances re-registered.")
                            }
                        }
                    }
                }
            }
        }
    }
}

