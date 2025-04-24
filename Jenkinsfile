pipeline {
    agent any

    environment {
        TARGET_GROUP_ARN = 'arn:aws:elasticloadbalancing:il-central-1:314525640319:targetgroup/tg-umat-haash/d6712b9674576b51'
        REGION           = 'il-central-1'
        INVENTORY_FILE   = 'inventory.ini'
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
                sshagent(credentials: ['aws']) {
                    script {
                        def instanceIds = env.INSTANCE_IDS.split(' ')
                        def instanceIps = instanceIds.collect { id ->
                            sh(
                                script: "aws ec2 describe-instances --instance-ids ${id} --region ${env.REGION} " +
                                        "--query 'Reservations[*].Instances[*].PrivateIpAddress' --output text",
                                returnStdout: true
                            ).trim()
                        }

                        writeFile file: env.INVENTORY_FILE, text: "[targets]\n" + instanceIps.join("\n")

                        // Optional: quick connectivity check
                        sh "ansible -i ${env.INVENTORY_FILE} targets -m ping -u ec2-user"

                        sh "ansible-playbook -i ${env.INVENTORY_FILE} playbook.yml -u ec2-user"
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

