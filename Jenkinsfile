pipeline {
    agent any

    environment {
        TARGET_GROUP_ARN = 'arn:aws:elasticloadbalancing:il-central-1:314525640319:targetgroup/tg-umat-haash/d6712b9676b51'
        REGION           = 'il-central-1'
        INVENTORY_FILE   = 'inventory.ini'
        SSH_CRED_ID      = 'aws'   // your SSH-key credential ID in Jenkins (SSH Username with private key)
    }

    stages {
        stage('Get Instances from Target Group') {
            steps {
                script {
                    def instancesJson = sh(
                        script: "aws elbv2 describe-target-health --target-group-arn ${TARGET_GROUP_ARN} --region ${REGION}",
                        returnStdout: true
                    ).trim()

                    def targetInstances = readJSON text: instancesJson
                    def instanceIds = targetInstances.TargetHealthDescriptions.collect { it.Target.Id }

                    writeFile file: 'instances.txt', text: instanceIds.join('\n')
                    env.INSTANCE_IDS = instanceIds.join(' ')
                }
            }
        }

        stage('Deregister Instances from Target Group') {
            steps {
                script {
                    def deregisterCmds = env.INSTANCE_IDS.split(' ').collect { "Id=${it}" }.join(' ')
                    sh "aws elbv2 deregister-targets --target-group-arn ${TARGET_GROUP_ARN} --targets ${deregisterCmds} --region ${REGION}"
                }
            }
        }

        stage('Run Ansible Playbook') {
            steps {
                script {
                    // 1. Resolve each instance's private IP
                    def instanceIps = env.INSTANCE_IDS.split(' ').collect { id ->
                        sh(
                            script: "aws ec2 describe-instances --instance-ids ${id} --region ${REGION} " +
                                    "--query 'Reservations[*].Instances[*].PrivateIpAddress' --output text",
                            returnStdout: true
                        ).trim()
                    }

                    // 2. Write inventory group [server] (matches your playbook's hosts)
                    writeFile file: INVENTORY_FILE, text: "[server]\n" + instanceIps.join("\n")
                    echo "Inventory:\n" + readFile(INVENTORY_FILE)
                }

                // 3. Inject SSH key via Credentials Binding and run playbook
                withCredentials([sshUserPrivateKey(
                    credentialsId: SSH_CRED_ID,
                    keyFileVariable: 'SSH_KEY_FILE',
                    usernameVariable: 'SSH_USER'
                )]) {
                    sh """
                      ansible-playbook -i ${INVENTORY_FILE} playbook.yml \
                        --user ${SSH_USER} \
                        --private-key ${SSH_KEY_FILE} \
                        --ssh-extra-args='-o StrictHostKeyChecking=no' \
                        -vvv
                    """
                }
            }
        }

        stage('Re-register Instances to Target Group') {
            steps {
                script {
                    def registerCmds = env.INSTANCE_IDS.split(' ').collect { "Id=${it}" }.join(' ')
                    sh "aws elbv2 register-targets --target-group-arn ${TARGET_GROUP_ARN} --targets ${registerCmds} --region ${REGION}"
                }
            }
        }
    }
}

