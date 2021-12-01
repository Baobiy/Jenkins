pipeline {
    agent {
        docker {
            image "harbor.smartx.com/smtxauto/${params.smtxauto_image}"
            alwaysPull true
            label "${params.jenkins_label}"
            args "--cpus 2 --privileged --device /dev/fuse --network host"
        }
    }
    options {
        timeout(time: 4, unit: 'HOURS') 
    }
    stages {
        

        stage('INIT') {
            steps {
                sh "${ENV_CMD} context create --meta_name ${BUILD_TAG} --console_loglevel INFO"
                sh "${ENV_CMD} context show"
                echo "load test information from config file to db"
                sh "${ENV_CMD} run load_config --conf_file ${ENV_CONFIG} --jinja2_variable 'cluster_type:${params.cluster_type}' --jinja2_variable 'nested_cluster_name:${params.nested_cluster_name}' --jinja2_variable 'node_num:${params.node_num}'  --jinja2_variable 'memory_size:${params.memory_size}'     --jinja2_variable 'cpu_num:${params.cpu_num}'  --jinja2_variable 'esxi_cpu_num:${params.esxi_cpu_num}'  --jinja2_variable 'esxi_memory_size:${params.esxi_memory_size}'  --jinja2_variable 'hostcluster_type:${params.hostcluster_type}' --jinja2_variable 'hostcluster_server:${params.hostcluster_server}' --jinja2_variable 'hostcluster_vcenter_server:${params.hostcluster_vcenter_server}' --jinja2_variable 'hostcluster_fisheye_user:${params.hostcluster_fisheye_user}' --jinja2_variable 'hostcluster_fisheye_password:${params.hostcluster_fisheye_password}' --jinja2_variable 'hostcluster_ssh_user:${params.hostcluster_ssh_user}' --jinja2_variable 'hostcluster_ssh_password:${params.hostcluster_ssh_password}' --jinja2_variable 'hostcluster_esxi_user:${params.hostcluster_esxi_user}' --jinja2_variable 'hostcluster_esxi_password:${params.hostcluster_esxi_password}' --jinja2_variable 'hostcluster_vcenter_user:${params.hostcluster_vcenter_user}' --jinja2_variable 'hostcluster_vcenter_password:${params.hostcluster_vcenter_password}' --jinja2_variable 'hostcluster_vcenter_datacenter:${params.hostcluster_vcenter_datacenter}'"
                sh "${ENV_CMD} context add_slack slack"
                sh "${ENV_CMD} context add_smtp smtp"
            }
        }

        stage('Import') {
            options {
                timeout(time: 2, unit: 'HOURS') 
            }
            parallel {
                stage('SmartxOS Image') {
                    steps {
                        sh "${ENV_CMD} cluster import_vm --cluster hostcluster --firmware '${params.firmware}' '${params.smartxos_image}'"
                    }
                }
                stage('ESXi Image') {
                    steps {
                        sh "${ENV_CMD} cluster import_vm  --cluster hostcluster '${params.esxi_image}'"
                    }
                }
            }
        }

        stage('Upload') {
            options {
                timeout(time: 2, unit: 'HOURS') 
            }
            parallel {
                stage('SmartxOS ISO') {
                    steps {
                        sh "${ENV_CMD} cluster upload_iso_to_datastore --cluster hostcluster '${params.smartxos_iso}'"
                    }
                }
                stage('ESXi ISO') {
                    steps {
                        sh "${ENV_CMD} cluster upload_iso_to_datastore  --cluster hostcluster '${params.esxi_iso}'"
                    }
                }
            }
        }

        stage('Create') {
            options {
                timeout(time: 2, unit: 'HOURS') 
            }
            parallel {
                stage('SmartxOS VM') {
                    steps {
                        script {
                                 if (params.cluster_type == "San"){
                                        sh "${ENV_CMD} cluster create_san_vm_image --cluster hostcluster --firmware '${params.firmware}'  '${params.smartxos_iso}'"
                                   }
                                   else {
                                        sh "${ENV_CMD} cluster create_halo_vm_image --cluster hostcluster --firmware '${params.firmware}' '${params.smartxos_iso}'"
                                    }
                                }
                    }
                }
                stage('ESXi VM') {
                    steps {
                        sh "${ENV_CMD} cluster create_esxi_vm_image  --cluster hostcluster '${params.esxi_iso}'"
                    }
                }
            }
        }

        stage('Export') {
            options {
                timeout(time: 2, unit: 'HOURS') 
            }
            parallel {
                stage('SmartxOS Image') {
                    steps {
                        sh "${ENV_CMD} cluster export_vm --cluster hostcluster '${params.smartxos_image}'"
                        sh "${ENV_CMD} cluster export_vm --cluster hostcluster '${params.smartxos_iso}'"
                    }
                }
                stage('ESXi Image') {
                    steps {
                        sh "${ENV_CMD} cluster export_vm  --cluster hostcluster '${params.esxi_image}'"
                        sh "${ENV_CMD} cluster export_vm  --cluster hostcluster '${params.esxi_iso}'"
                    }
                }
            }
        }

        stage('Deploy') {
            options {
                timeout(time: 2, unit: 'HOURS') 
            }
            steps {
                script {
                    def cmd = "${env.ENV_CMD} cluster create_nested_cluster --cluster nestedcluster --smartxos_iso='${params.smartxos_iso}' --esxi_iso='${params.esxi_iso}' --smartxos_image='${params.smartxos_image}' --esxi_image='${params.esxi_image}' --xen_version='${params.xen_version}' --cache_disk_num='${params.cache_disk_num}' --deploy_cluster='${params.deploy_cluster}' --firmware '${params.firmware}' "
                    if (params.vhost_enabled){
                        cmd = "${cmd}   --vhost_enabled"
                    } 
                    if (params.is_metro_availability){
                        cmd = "${cmd}  --is_metro_availability"
                    }
                    echo "${cmd}"
                    sh "${cmd}"
                }
            }
        }
        stage('Restart') {
            options {
                timeout(time: 2, unit: 'HOURS') 
            }
            steps {
                 script {
                    if (params.vhost_enabled){
                        sh "${ENV_CMD} cluster stop_nested_cluster --host_cluster hostcluster --kind '${params.cluster_type}' nestedcluster"
                        sh "${ENV_CMD} cluster start_nested_cluster --host_cluster hostcluster --kind '${params.cluster_type}' nestedcluster"
                    } 
                }
            }
        }

    }

    post {
        always {
            sh "${ENV_CMD} cluster send_nested_cluster_info --cluster nestedcluster  --slack_channel='${params.slack_channel}' --slack_users='${params.slack_users}'  --mail_to='${params.mail_to}' --job_url='${JOB_URL}' --deploy_cluster='${params.deploy_cluster}' || echo 'Done'"
        }

    }
    environment {
        JOB_URL = "${BUILD_URL}"
        ENV_CONFIG = "nested_cluster-on-specific-cluster.yaml"
        SA_SOCKET_CONNECT_TIMEOUT = 20
        ENV_CMD = "smtxauto "
    }
    parameters {
        choice(name: 'cluster_type', choices: ['ELF', 'VMware', 'San', 'Xenserver_No_Hyper', 'VMware_No_Hyper'], description: '嵌套集群的平台类型')
        choice(name: 'xen_version', choices: ['xenserver7.1', 'xenserver', 'xenserver7'], description: '部署xenserver 集群时，xen 的版本号')
        choice(name: 'deploy_cluster', choices: ['Yes', 'No'], description: '是否需要部署集群，如果选择不部署集群则只创建 scvm 节点。如果 scvm 节点是用于在界面部署集群和添加节点，缓存盘个数必须选择2个，因为界面上做了限制，1 个缓存盘不允许部署集群和添加节点')
        booleanParam(name: 'vhost_enabled', defaultValue: false , description: '集群是否开启 vhost, 仅 ELF 集群支持开启 vhost')
        booleanParam(name: 'is_metro_availability', defaultValue: false , description: '是否部署双活集群')
        choice(name: 'jenkins_label', choices: ["bj-idc","bj-lab"], description: 'Jenkins node label')
        string(name: 'nested_cluster_name', defaultValue: 'nestedcluster', description: '嵌套集群 scvm vm 的名称前缀，也是部署嵌套集群的名称，建议包含自己的名字')
        choice(name: 'hostcluster_type', choices: ['ELF', 'VMware'], description: '创建嵌套环境的物理集群的平台类型，ELF 或者VMware')
        string(name: 'hostcluster_server', defaultValue: '', description: '物理集群的任一节点 ip')
        string(name: 'hostcluster_vcenter_server', defaultValue: '', description: 'VMware 物理集群的 Vcenter ip, ELF 物理集群忽略此参数')
        string(name: 'hostcluster_fisheye_user', defaultValue: 'root', description: '物理集群的Fisheye user')
        string(name: 'hostcluster_fisheye_password', defaultValue: '111111', description: '物理集群的 Fisheye password')
        string(name: 'hostcluster_ssh_user', defaultValue: 'smtxauto', description: '物理集群 scvm 节点的 ssh user')
        string(name: 'hostcluster_ssh_password', defaultValue: 'HC!r0cks', description: '物理集群 scvm 节点的ssh password')
        string(name: 'hostcluster_esxi_user', defaultValue: 'root', description: 'VMware 物理集群的 Esxi 主机用户名， ELF 物理集群忽略此参数')
        string(name: 'hostcluster_esxi_password', defaultValue: 'abcd1234', description: 'VMware 物理集群的 Esxi 主机密码， ELF 物理集群忽略此参数')
        string(name: 'hostcluster_vcenter_user', defaultValue: 'vsphere.local\\\\Administrator', description: 'VMware 物理集群的 Vcenter用户名， ELF 物理集群忽略此参数')
        string(name: 'hostcluster_vcenter_password', defaultValue: '!QAZ2wsx', description: 'VMware 物理集群的 Vcenter 密码， ELF 物理集群忽略此参数')
        string(name: 'hostcluster_vcenter_datacenter', defaultValue: 'QE-DOGFOOD', description: 'VMware 物理集群的 Vcenter Datacenter名称， ELF 物理集群忽略此参数')
        string(name: 'smartxos_image', defaultValue:'', description: '创建嵌套集群所使用的 smartxos image url。 已有 smartxos image 存放路径: http://192.168.67.2/distros/isostore/smartxos_image/  或者  http://192.168.17.20/smartxos_image/  (使用所选物理集群对应平台下的镜像，优先使用与物理集群同一网段的 image)')
        string(name: 'smartxos_iso', defaultValue: '', description: '创建嵌套集群所使用的 smartxos iso url , smartxos_image 和 smartxos_iso 提供任意一个参数即可，优先使用 smartxos_image。已发布的 smartxos iso 存放路径: http://192.168.17.20/repo/pub/smartxos/iso/ . 未发布的 smartxos iso 存放路径： http://192.168.17.20/iso/smartxos/')
        choice(name: 'esxi_image', choices: ['','http://192.168.67.2/distros/isostore/vmimages/esxi/VMware-VMvisor-Installer-6.0.0-2494585.x86_64.ova','http://192.168.67.2/distros/isostore/vmimages/esxi/VMware-VMvisor-Installer-6.0.0.update03-5572656.x86_64.ova', 'http://192.168.67.2/distros/isostore/vmimages/esxi/VMware-VMvisor-Installer-6.5.0-4564106.x86_64.ova', 'http://192.168.67.2/distros/isostore/vmimages/esxi/VMware-VMvisor-Installer-6.5.0.update02-8294253.x86_64.ova', 'http://192.168.67.2/distros/isostore/vmimages/esxi/VMware-VMvisor-Installer-6.7.0-8169922.x86_64.ova', 'http://192.168.17.20/esxi_image/VMware-VMvisor-Installer-6.0.0-2494585.x86_64.ova', 'http://192.168.17.20/esxi_image/VMware-VMvisor-Installer-6.0.0.update03-5572656.x86_64.ova', 'http://192.168.17.20/esxi_image/VMware-VMvisor-Installer-6.5.0-4564106.x86_64.ova', 'http://192.168.17.20/esxi_image/VMware-VMvisor-Installer-6.5.0.update02-8294253.x86_64.ova', 'http://192.168.17.20/esxi_image/VMware-VMvisor-Installer-6.7.0-8169922.x86_64.ova'], description: '创建 vmware 集群使用的 esxi image url, 优先使用与物理集群同一网段的 image')
        string(name: 'esxi_iso', defaultValue: '', description: '创建 vmware 集群的 esxi iso url, esxi_image 和 esxi_iso 提供任意一个参数即可，优先使用 esxi_image。 esxi iso 存放路径 ：http://192.168.67.2/distros/isostore/VMware/')
        string(name: 'node_num', defaultValue: '3', description: '嵌套集群的节点个数，部署双活集群时，为仲裁节点以外的节点个数')
        string(name: 'memory_size', defaultValue: '16G', description: '嵌套集群每个节点的内存')
        string(name: 'esxi_memory_size', defaultValue: '20G', description: 'vmware 嵌套集群每个esxi 节点的内存')
        string(name: 'cpu_num', defaultValue: '8', description: '嵌套集群每个scvm节点的cpu 核数')
        string(name: 'esxi_cpu_num', defaultValue: '10', description: '嵌套集群每个esxi 节点的cpu 核数')
        choice(name: 'cache_disk_num', choices: [1, 2], description: '嵌套集群每个节点缓存盘个数，默认1个，非特殊需求不要选2个，因为耗时较长')
        choice(name: 'firmware', choices: ["BIOS", "UEFI"], description: '嵌套集群节点的固件类型, ARM 集群只支持 UEFI')
        string(name: 'slack_channel', defaultValue: 'nested-cluster', description: '发送通知的 slack channel 名称')
        string(name: 'slack_users', defaultValue: 'liuyang', description: '需要 @ 的 slack 用户名, 多个用户使用逗号 "," 分割')
        string(name: 'mail_to', defaultValue: 'liuyang@smartx.com', description: '发送通知的邮件名称， 多个邮件使用逗号 "," 分割')
        string(name: 'smtxauto_image', defaultValue: 'smtxauto:v0.1.14', description: 'smtxauto image version, 默认即可')
    }
}