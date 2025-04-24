pipeline {
    agent any

    environment {
        TARGET_GROUP_ARN = 'arn:aws:elasticloadbalancing:il-central-1:314525640319:targetgroup/tg-umat-haash/d6712b9674576b51'
        REGION = 'il-central-1'
        INVENTORY_FILE = 'inventory.ini'
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

                    // Store in a file or env var
                    writeFile file: 'instances.txt', text: instanceIds.join('\n')
                    env.INSTANCE_IDS = instanceIds.join(' ')
                }
            }
        }

        stage('Deregister Instances from Target Group') {
            steps {
                script {
                    def deregisterCmds = env.INSTANCE_IDS.split(' ').collect {
                        "Id=${it}"
                    }.join(' ')

                    sh "aws elbv2 deregister-targets --target-group-arn ${env.TARGET_GROUP_ARN} --targets ${deregisterCmds} --region ${env.REGION}"
                }
            }
        }

        stage('Run Ansible Playbook') {
            steps {
                script {
                    def instanceIds = env.INSTANCE_IDS.split(' ')
                    def instanceIps = []

                    for (id in instanceIds) {
                        def ip = sh(script: "aws ec2 describe-instances --instance-ids ${id} --region ${env.REGION} --query 'Reservations[*].Instances[*].PrivateIpAddress' --output text", returnStdout: true).trim()
                        instanceIps.add(ip)
                    }

                    def inventory = "[targets]\n" + instanceIps.join("\n")
                    writeFile file: env.INVENTORY_FILE, text: inventory

                    echo "Running Ansible playbook..."
                    sh "ansible-playbook -i ${env.INVENTORY_FILE} playbook.yml"
                }
            }
        }

        stage('Re-register Instances to Target Group') {
            steps {
                script {
                    echo "Re-registering instances..."
                    def registerCmds = env.INSTANCE_IDS.split(' ').collect {
                        "Id=${it}"
                    }.join(' ')

                    sh "aws elbv2 register-targets --target-group-arn ${env.TARGET_GROUP_ARN} --targets ${registerCmds} --region ${env.REGION}"
                }
            }
        }
    }
}
