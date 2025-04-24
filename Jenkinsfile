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
                    def instanceIds     = targetInstances.TargetHealthDescriptions*.Target.Id

                    writeFile file: 'instances.txt', text: instanceIds.join('\n')
                    env.INSTANCE_IDS = instanceIds.join(' ')
                }
            }
        }

        stage('Deregister Instances from Target Group') {
            steps {
                script {
                    def deregisterCmds = env.INSTANCE_IDS.split(' ')
                                          .collect { "Id=${it}" }
                                          .join(' ')
                    sh """
                      aws elbv2 deregister-targets \
                        --target-group-arn ${env.TARGET_GROUP_ARN} \
                        --targets ${deregisterCmds} \
                        --region ${env.REGION}
                    """
                }
            }
        }

        stage('Run Ansible Playbook') {
            steps {
                // bind your 'aws' SSH key into $SSH_KEY_PATH
                withCredentials([sshUserPrivateKey(
                  credentialsId:   'aws',
                  keyFileVariable: 'SSH_KEY_PATH',
                  passphraseVariable: '',   // if encrypted, set this
                  usernameVariable:  ''     // not needed here
                )]) {
                    script {
                        // build list of private IPs
                        def instanceIps = env.INSTANCE_IDS.split(' ').collect { id ->
                            sh(
                              script: "aws ec2 describe-instances --instance-ids ${id} " +
                                      "--region ${env.REGION} " +
                                      "--query 'Reservations[*].Instances[*].PrivateIpAddress' " +
                                      "--output text",
                              returnStdout: true
                            ).trim()
                        }

                        writeFile file: env.INVENTORY_FILE, text: "[targets]\n" + instanceIps.join("\n")

                        // 🔍 quick connectivity check
                        sh """
                          ansible -i ${env.INVENTORY_FILE} targets \
                            -m ping \
                            -u ec2-user \
                            --private-key=${SSH_KEY_PATH}
                        """

                        // 🚀 run the actual playbook
                        sh """
                          ansible-playbook \
                            -i ${env.INVENTORY_FILE} \
                            -u ec2-user \
                            --private-key=${SSH_KEY_PATH} \
                            playbook.yml
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            echo "🔄 Re-registering targets back into the ALB target group…"
            script {
                if (!env.INSTANCE_IDS) {
                    error "No INSTANCE_IDS found—skipping re-register."
                }
                def registerCmds = env.INSTANCE_IDS.split(' ')
                                       .collect { "Id=${it}" }
                                       .join(' ')
                sh """
                  aws elbv2 register-targets \
                    --target-group-arn ${env.TARGET_GROUP_ARN} \
                    --targets ${registerCmds} \
                    --region ${env.REGION}
                """
            }
        }
    }
}

