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
                    // Fetch the list of instance IDs currently in the target group
                    def instancesJson = sh(
                        script: "aws elbv2 describe-target-health --target-group-arn ${env.TARGET_GROUP_ARN} --region ${env.REGION}",
                        returnStdout: true
                    ).trim()
                    def targetInstances = readJSON text: instancesJson
                    def instanceIds     = targetInstances.TargetHealthDescriptions*.Target.Id

                    // Persist for later stages
                    writeFile file: 'instances.txt', text: instanceIds.join('\n')
                    env.INSTANCE_IDS = instanceIds.join(' ')
                }
            }
        }

        stage('Deregister Instances from Target Group') {
            steps {
                script {
                    def deregisterArgs = env.INSTANCE_IDS
                        .split(' ')
                        .collect { "Id=${it}" }
                        .join(' ')
                    sh """
                      aws elbv2 deregister-targets \
                        --target-group-arn ${env.TARGET_GROUP_ARN} \
                        --targets ${deregisterArgs} \
                        --region ${env.REGION}
                    """
                }
            }
        }

        stage('Run Ansible Playbook via SSM') {
            steps {
                script {
                    // Build an inventory file listing instance IDs with the SSM connection
                    def inventories = env.INSTANCE_IDS
                        .split(' ')
                        .collect { id -> "${id} ansible_connection=community.aws.aws_ssm" }
                        .join('\n')
                    writeFile file: env.INVENTORY_FILE, text: "[targets]\n" + inventories

                    // Execute the playbook
                    sh """
                      ansible-playbook \
                        -i ${env.INVENTORY_FILE} \
                        playbook.yml
                    """
                }
            }
        }
    }

    post {
        always {
            echo "🔄 Re-registering instances back into the target group…"
            script {
                if (!env.INSTANCE_IDS) {
                    echo "⚠️ No instances were recorded; skipping re-register."
                } else {
                    def registerArgs = env.INSTANCE_IDS
                        .split(' ')
                        .collect { "Id=${it}" }
                        .join(' ')
                    sh """
                      aws elbv2 register-targets \
                        --target-group-arn ${env.TARGET_GROUP_ARN} \
                        --targets ${registerArgs} \
                        --region ${env.REGION}
                    """
                }
            }
        }
    }
}

