pipeline {
    agent any

    environment {
        TARGET_GROUP_ARN = 'arn:aws:elasticloadbalancing:il-central-1:314525640319:targetgroup/tg-umat-haash/d6712b9674576b51'
        REGION           = 'il-central-1'
        INVENTORY_FILE   = 'inventory.ini'
        SSH_CRED_ID      = 'aws'  // your SSH-key credential ID in Jenkins
    }

    stages {
        stage('Get Instances from Target Group') {
            steps {
                script {
                    def instancesJson = sh(
                        script: "aws elbv2 describe-target-health --target-group-arn ${env.TARGET_GROUP_ARN} --region ${env.REGION}",
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
                    sh "aws elbv2 deregister-targets --target-group-arn ${env.TARGET_GROUP_ARN} --targets ${deregisterCmds} --region ${env.REGION}"
                }
            }
        }

        stage('Run Ansible Playbook') {
            steps {
                script {
                    // Resolve private IPs for each instance ID
                    def instanceIps = env.INSTANCE_IDS.split(' ').collect { id ->
                        sh(
                            script: "aws ec2 describe-instances --instance-ids ${id} --region ${env.REGION} --query 'Reservations[*].Instances[*].PrivateIpAddress' --output text",
                            returnStdout: true
                        ).trim()
                    }

                    // Build inventory group [server] to match your playbook
                    writeFile file: env.INVENTORY_FILE, text: "[server]\n" + instanceIps.join("\n")
                    echo "Inventory:\n${readFile(env.INVENTORY_FILE)}"

                    // Run playbook with Jenkins-loaded SSH key
                    sshagent (credentials: [env.SSH_CRED_ID]) {
                        sh """
                          ansible-playbook -i ${env.INVENTORY_FILE} playbook.yml \
                            --user ubuntu \
                            --ssh-extra-args='-o StrictHostKeyChecking=no' -vvv
                        """
                    }
                }
            }
        }

        stage('Re-register Instances to Target Group') {
            steps {
                script {
                    def registerCmds = env.INSTANCE_IDS.split(' ').collect { "Id=${it}" }.join(' ')
                    sh "aws elbv2 register-targets --target-group-arn ${env.TARGET_GROUP_ARN} --targets ${registerCmds} --region ${env.REGION}"
                }
            }
        }
    }
}

