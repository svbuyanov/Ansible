# CloudWatchAgentInstallationWithAnsible

This repository includes Ansible-Playbook to install and start CloudWatch Agent to a Linux based EC2 instance and a configuration JSON file to collect some crucial metrics from it.

Please check your region and environments, change the variables in script if needed.

## For further information refer:
http://code-ogre.com/2018/06/11/cloudwatch-agent-installation-to-ec2-instances-with-ansible/

## Ansible-Playbook:

```yaml
---

- hosts: xxx.xxx.xxx.xxx
  remote_user: root
  gather_facts: true
  tasks:
    - name: Check if Cloudwatch Agent is Installed Already
      command: status amazon-cloudwatch-agent
      register: init_status_result
      ignore_errors: yes

    - debug:
      var: init_status_result.stderr
      verbosity: 2

    - name: Create Directory for Downloading Cloudwatch Agent .zip
      file:
        path: /opt/aws/amazon-cloudwatch-zip
        state: directory
        owner: root
        group: root
        mode: 0755
        recurse: no
      when: init_status_result.stderr | search("Unknown job")

    - name: Download Latest Version of Amazon Cloudwatch Agent
      get_url:
        url: "https://s3.amazonaws.com/amazoncloudwatch-agent/linux/amd64/latest/AmazonCloudWatchAgent.zip"
        dest: /opt/aws/amazon-cloudwatch-zip/AmazonCloudWatchAgent.zip
        mode: 0755
      when: init_status_result.stderr | search("Unknown job")

    - name: Unzip Cloudwatch Download File
      unarchive:
        remote_src: yes
        src: /opt/aws/amazon-cloudwatch-zip/AmazonCloudWatchAgent.zip
        dest: /opt/aws/amazon-cloudwatch-zip
      when: init_status_result.stderr | search("Unknown job")

    - name: Execute the Installation Script
      command: /opt/aws/amazon-cloudwatch-zip/install.sh
      args:
        chdir: /opt/aws/amazon-cloudwatch-zip
      when: init_status_result.stderr | search("Unknown job")

    - name: Transfer Cloudwatch Common Configuration(Proxies...) File
      copy:
        src: /files/common-config.toml
        dest: /opt/aws/amazon-cloudwatch-agent/etc
        owner: ec2-user
        group: ec2-user
        mode: 0755
      when: init_status_result.stderr | search("Unknown job")

    - name: Transfer Cloudwatch Configuration File
      copy:
        src: /files/amazon-cloudwatch-agent.json
        dest: /opt/aws/amazon-cloudwatch-agent/etc
        owner: ec2-user
        group: ec2-user
        mode: 0755

    - name: Stop Amazon Cloudwatch Agent
      command: stop amazon-cloudwatch-agent
      ignore_errors: yes

    - name: Start Amazon Cloudwatch Agent
      command: start amazon-cloudwatch-agent
```

## Configuration JSON to put under /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json

```json
{
  "agent": {
    "metrics_collection_interval": 30,
    "logfile": "/opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log"
  },
  "metrics": {
    "metrics_collected": {
      "cpu": {
        "resources": [
          "*"
        ],
        "measurement": [
          {"name": "cpu_usage_idle", "rename": "CPU_USAGE_IDLE", "unit": "Percent"},
          {"name": "cpu_usage_iowait", "rename": "CPU_USAGE_IOWAIT", "unit": "Percent"},
          {"name": "cpu_time_idle", "rename": "CPU_TIME_IDLE", "unit": "Percent"},
          {"name": "cpu_time_iowait", "rename": "CPU_TIME_IOWAIT", "unit": "Percent"}
        ]
      },
      "disk": {
        "resources": [
          "/"
        ],
        "measurement": [
          {"name": "disk_free", "rename": "DISK_FREE", "unit": "Gigabytes"},
          {"name": "disk_inodes_free", "rename": "DISK_INODES_FREE", "unit": "Count"},
          {"name": "disk_inodes_total", "rename": "DISK_INODES_TOTAL", "unit": "Count"},
          {"name": "disk_inodes_used", "rename": "DISK_INODES_USED", "unit": "Count"}
        ]
      },
      "diskio": {
        "resources": [
          "*"
        ],
        "measurement": [
          {"name": "diskio_iops_in_progress", "rename": "DISKIO_IOPS_IN_PROGRESS", "unit": "Megabytes"},
          {"name": "diskio_read_time", "rename": "DISKIO_READ_TIME", "unit": "Megabytes"},
          {"name": "diskio_write_time", "rename": "DISKIO_WRITE_TIME", "unit": "Megabytes"}
        ]
      },
      "mem": {
        "measurement": [
          {"name": "mem_free", "rename": "MEM_FREE", "unit": "Megabytes"},
          {"name": "mem_total", "rename": "MEM_TOTAL", "unit": "Megabytes"},
          {"name": "mem_used", "rename": "MEM_USED", "unit": "Megabytes"}
        ]
      },
      "net": {
        "resources": [
          "eth0"
        ],
        "measurement": [
          {"name": "net_bytes_recv", "rename": "NET_BYTES_RECV", "unit": "Bytes"},
          {"name": "net_bytes_sent", "rename": "NET_BYTES_SENT", "unit": "Bytes"}
        ]
      },
      "netstat": {
        "measurement": [
          {"name": "netstat_tcp_listen", "rename": "NETSTAT_TCP_LISTEN", "unit": "Count"},
          {"name": "netstat_tcp_syn_sent", "rename": "NETSTAT_TCP_SYN_SENT", "unit": "Count"},
          {"name": "netstat_tcp_established", "rename": "NETSTAT_TCP_ESTABLISHED", "unit": "Count"}
        ]
      },
      "processes": {
        "measurement": [
          {"name": "processes_blocked", "rename": "PROCESSES_BLOCKED", "unit": "Count"},
          {"name": "processes_running", "rename": "PROCESSES_RUNNING", "unit": "Count"},
          {"name": "processes_zombies", "rename": "PROCESSES_ZOMBIES", "unit": "Count"}
        ]
      }
    },
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}",
      "InstanceType": "${aws:InstanceType}"
    },
    "aggregation_dimensions" : [["InstanceId", "InstanceType"]]
  }
}
```
