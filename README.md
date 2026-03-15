Ansible AI Agent Demo – Updated Version

This project demonstrates an AI-assisted infrastructure automation workflow built using:

Ansible for automation

Streamlit for the user interface

Ollama LLM (phi3:mini) for AI reasoning

Python for orchestration

The system allows users to:

Run infrastructure validation playbooks.

Ask an AI agent to analyze failures.

Automatically generate operational reports.

Run additional verification playbooks via natural language prompts.

Architecture Overview
Component	Description
Ansible Controller VM	Azure Linux VM running Ubuntu 24.04 (Standard_E2s_v4, 2 vCPU, 16GB RAM). Hosts Ansible, Streamlit and Ollama.
Ansible Version	Ansible Core 2.16.3
Python Environment	Isolated Python virtual environment
Streamlit	Version 1.54 – provides web UI
LLM Model	Ollama phi3:mini for AI reasoning and analysis
Managed Hosts	Two Azure Debian 12 VMs accessed via SSH
Artifacts Directory	Stores JSON results from playbooks
Folder Structure
/home/pearlzfan/

├── ansible/
│   ├── connection_validator.yml
│   ├── check_server_name.yml
│   ├── inventory.ini
│
├── ansible_web/
│   ├── app.py
│   └── venv/
│
├── artifacts/
│   └── raw_runs/

Directory purpose:

Folder	Purpose
ansible/	Automation playbooks and inventory
ansible_web/	Streamlit AI agent application
artifacts/raw_runs/	JSON outputs produced by playbooks
System Preparation
Update OS
sudo apt update && sudo apt upgrade -y
Install Python
sudo apt install -y python3 python3-venv python3-distutils python3-pip
python3 -m pip install --upgrade pip
Install Ansible
python3 -m pip install --user ansible-core==2.16.3
SSH Configuration

Generate SSH key:

ssh-keygen -t rsa -b 4096 -f ~/ansible-controller_key.pem
chmod 600 ~/ansible-controller_key.pem

Copy key to managed hosts:

ssh-copy-id -i ~/ansible-controller_key.pem pearlzfan@10.0.0.7
ssh-copy-id -i ~/ansible-controller_key.pem pearlzfan@10.0.0.8
Ansible Inventory

inventory.ini

[linux]
host-vm1 ansible_host=10.0.0.7
host-vm2 ansible_host=10.0.0.8

[all:vars]
ansible_user=pearlzfan
ansible_ssh_private_key_file=/home/pearlzfan/ansible-controller_key.pem
Playbook 1 – Connection Validator

This playbook checks whether each server is reachable via SSH and stores the results as a JSON artifact.

connection_validator.yml

The playbook:

pings each host

collects connection status

stores structured results

Example JSON artifact:

artifacts/raw_runs/connection_validator_20260315_110230.json

Example content:

[
  {
    "hostname": "host-vm1",
    "ip": "10.0.0.7",
    "success": true,
    "details": "pong"
  },
  {
    "hostname": "host-vm2",
    "ip": "10.0.0.8",
    "success": false,
    "details": "Host unreachable"
  }
]
Playbook 2 – Hostname Comparison
check_server_name.yml

Purpose:

Compare:

hostname defined in Ansible inventory

actual hostname configured on the server

Enhancements included:

status classification

remediation suggestion

Status Types
Status	Meaning
MATCH	Inventory hostname equals real hostname
MISMATCH	Names differ
ERROR	Hostname could not be retrieved

Example artifact:

artifacts/raw_runs/check_server_name_20260315_114100.json

Example output:

[
  {
    "inventory_hostname": "host-vm1",
    "ip_address": "10.0.0.7",
    "real_hostname": "vm1",
    "status": "MISMATCH",
    "agent_suggestion": "Update inventory hostname or rename server"
  },
  {
    "inventory_hostname": "host-vm2",
    "ip_address": "10.0.0.8",
    "real_hostname": "host-vm2",
    "status": "MATCH",
    "agent_suggestion": "No action required"
  }
]
Streamlit AI Agent

File:

ansible_web/app.py

The application provides an interface where the user can send natural language instructions to the AI agent.

Example prompt:

Prepare CSV report of hosts that failed connectivity check
AI Capabilities

The AI agent performs three tasks.

1 – Intent Detection

The LLM analyzes the user request and determines the action.

Possible intents:

FAILED_HOSTS_REPORT
RUN_HOSTNAME_CHECK
UNKNOWN
2 – Failure Analysis

When a host fails the connection validator, the LLM analyzes the result and generates:

Root cause

Suggested remediation

Example output:

Root Cause: SSH service not reachable on host
Suggested Fix: Verify network connectivity and ensure SSH daemon is running
3 – Automated Reporting

The agent generates CSV reports based on playbook artifacts.

Generated Reports
Connectivity Failure Report

Generated after analyzing failed hosts.

Columns:

| Hostname | IP Address | Last Playbook Run | Reason of Fail | Suggested Fix |

Example:

Hostname	IP	Last Run	Reason	Suggested Fix
host-vm2	10.0.0.8	20260315_110230	SSH connection failed	Verify SSH service and firewall
Hostname Comparison Report

Generated after running hostname comparison playbook.

Columns:

| Ansible Inventory Hostname | Real Hostname | Status | Agent Suggestion |

Example:

Inventory Hostname	Real Hostname	Status	Suggestion
host-vm1	vm1	MISMATCH	Update inventory hostname
host-vm2	host-vm2	MATCH	No action required
Artifact Cleanup

Each time the Streamlit application starts, it performs automatic cleanup.

The system keeps:

only the newest JSON artifact

only the newest CSV report

Older files are deleted to prevent artifact accumulation.

Demo Workflow
Step 1 – Run connectivity validation
ansible-playbook -i ~/ansible/inventory.ini ~/ansible/connection_validator.yml

This creates a JSON artifact.

Step 2 – Ask AI for failure analysis

Open the Streamlit interface and enter:

Prepare CSV report of hosts that failed connectivity check

The AI agent:

Loads the latest connection artifact.

Identifies failed hosts.

Uses LLM reasoning to analyze failures.

Generates a CSV report.

Step 3 – Run hostname comparison

User prompt:

Check real hostnames of servers

The AI agent:

Detects user intent.

Runs the check_server_name.yml playbook.

Loads the resulting JSON artifact.

Generates a hostname comparison report.

Key Features Demonstrated

This demo showcases:

✔ Infrastructure validation using Ansible
✔ AI-driven failure analysis
✔ Natural language automation interface
✔ Automated reporting
✔ Integration of automation and LLM reasoning

The system illustrates how AI can assist infrastructure engineers by analyzing operational data and generating actionable insights.
