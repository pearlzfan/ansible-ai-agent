# Ansible AI Agent Demo – Enhanced Version

This repository demonstrates an enhanced Ansible AI Agent using Ansible, Streamlit, and Ollama LLM to automate infrastructure analysis, playbook execution, and CSV reporting.

## Architecture Overview

| Component             | Description                                                                                                      |
| --------------------- | ---------------------------------------------------------------------------------------------------------------- |
| Ansible Controller VM | Azure Linux VM (Ubuntu 24.04, Standard_E2s_v4, 2 vCPU, 16 GB RAM). Hosts Ansible, Python, Streamlit, and Ollama. |
| Ansible Version       | Core 2.16.3                                                                                                      |
| Python & venv         | Isolated Python virtual environment for dependencies                                                             |
| Streamlit             | Version 1.54.0, provides web interface for user prompts and CSV reports                                          |
| LLM Model             | Ollama phi3:mini for AI analysis of failed hosts                                                                 |
| Managed Hosts         | 2 Azure VMs (Debian 12, Standard_D2s_v3), accessed via SSH by Ansible                                            |

## Folder Structure

```
/home/pearlzfan/
├── ansible/
│   ├── connection_validator.yml
│   ├── check_server_name.yml
│   ├── inventory.ini
├── ansible_web/
│   ├── app.py
│   └── venv/
├── artifacts/
│   └── raw_runs/
```

`artifacts/raw_runs` contains standardized JSON outputs for both playbooks.

## System Preparation

### Update system

```
sudo apt update && sudo apt upgrade -y
```

### Install Python and dependencies

```
sudo apt install -y python3 python3-venv python3-distutils python3-pip
python3 -m pip install --upgrade pip
```

### Install Ansible Core

```
python3 -m pip install --user ansible-core==2.16.3
```

### Setup SSH keys for managed hosts

```
ssh-keygen -t rsa -b 4096 -f ~/ansible-controller_key.pem
chmod 600 ~/ansible-controller_key.pem
ssh-copy-id -i ~/ansible-controller_key.pem pearlzfan@10.0.0.7
ssh-copy-id -i ~/ansible-controller_key.pem pearlzfan@10.0.0.8
```

## Ansible Inventory (`inventory.ini`)

```
[linux]
host-vm1 ansible_host=10.0.0.7
host-vm2 ansible_host=10.0.0.8

[all:vars]
ansible_user=pearlzfan
ansible_ssh_private_key_file=/home/pearlzfan/ansible-controller_key.pem
```

## Connection Validator Playbook (`connection_validator.yml`)

```yaml
---
- name: Connection Validator — all hosts
  hosts: localhost
  gather_facts: no
  vars:
    target_group: linux
    run_timestamp: "{{ lookup('pipe','date +%Y%m%d_%H%M%S') }}"

  tasks:
    - name: Ensure raw_runs directory exists
      file:
        path: "{{ playbook_dir }}/../artifacts/raw_runs"
        state: directory
        mode: '0755'

    - name: Initialize results list
      set_fact:
        all_results: []

    - name: Ping each host
      ansible.builtin.ping:
      delegate_to: "{{ item }}"
      register: ping_result
      ignore_unreachable: yes
      ignore_errors: yes
      loop: "{{ groups[target_group] }}"
      loop_control:
        loop_var: item

    - name: Append ping results to all_results
      set_fact:
        all_results: "{{ all_results + [ {
          'hostname': item.item,
          'ip': hostvars[item.item].ansible_host | default('unknown'),
          'os_family': hostvars[item.item].ansible_os_family | default('unknown'),
          'distribution': hostvars[item.item].ansible_distribution | default('unknown'),
          'success': (item.ping is defined and item.ping == 'pong'),
          'details': (item.ping if item.ping is defined and item.ping == 'pong' else 'Host unreachable or SSH failure')
        } ] }}"
      loop: "{{ ping_result.results }}"
      loop_control:
        loop_var: item

    - name: Write JSON artifact
      copy:
        content: "{{ all_results | to_nice_json }}"
        dest: "{{ playbook_dir }}/../artifacts/raw_runs/connection_validator_{{ run_timestamp }}.json"
```

## Server Name Playbook (`check_server_name.yml`)

```yaml
---
- name: Check Server Hostname
  hosts: linux
  gather_facts: no
  vars:
    run_timestamp: "{{ lookup('pipe','date +%Y%m%d_%H%M%S') }}"

  tasks:
    - name: Ensure artifacts directory exists
      file:
        path: "{{ playbook_dir }}/../artifacts/raw_runs"
        state: directory
        mode: '0755'

    - name: Initialize results
      set_fact:
        all_results: []

    - name: Collect hostname
      command: hostname
      register: hostname_output
      ignore_errors: yes

    - name: Append results
      set_fact:
        all_results: "{{ all_results + [{
          'playbook': 'check_server_name',
          'hostname': inventory_hostname,
          'ip': ansible_host,
          'success': hostname_output.rc == 0,
          'server_name': hostname_output.stdout | default('unknown'),
          'details': hostname_output.stderr | default('ok')
        }] }}"

    - name: Write JSON artifact
      copy:
        content: "{{ all_results | to_nice_json }}"
        dest: "{{ playbook_dir }}/../artifacts/raw_runs/check_server_name_{{ run_timestamp }}.json"
```

## Streamlit App (`app.py`)

```python
import streamlit as st
import json
import os
import subprocess
import csv
from datetime import datetime
import concurrent.futures

ARTIFACTS_DIR = "/home/pearlzfan/artifacts/raw_runs"
ANSIBLE_DIR = "/home/pearlzfan/ansible"
INVENTORY = f"{ANSIBLE_DIR}/inventory.ini"
CSV_DIR = "/home/pearlzfan/ansible_web"
OLLAMA_PATH = "/usr/local/bin/ollama"
OLLAMA_MODEL = "phi3:mini"
OLLAMA_TIMEOUT = 300

def cleanup_old_files():
    def keep_newest(directory, extension):
        files = [os.path.join(directory, f) for f in os.listdir(directory) if f.endswith(extension)]
        if len(files) <= 1:
            return
        files.sort(key=os.path.getmtime, reverse=True)
        for f in files[1:]:
            try:
                os.remove(f)
            except:
                pass
    keep_newest(CSV_DIR, ".csv")
    keep_newest(ARTIFACTS_DIR, ".json")

cleanup_old_files()

def warm_up_model():
    try:
        subprocess.run([OLLAMA_PATH, "run", OLLAMA_MODEL], input="Hello", text=True, capture_output=True, timeout=60)
    except:
        pass

warm_up_model()

st.set_page_config(page_title="Ansible AI Agent", layout="wide")
st.title("Ansible AI Agent")

user_prompt = st.text_area("Enter instruction for the AI agent", placeholder="Example: Prepare CSV report of failed servers")
run_agent = st.button("Run AI Agent")

def run_playbook(playbook, limit_hosts=None):
    playbook_path = f"{ANSIBLE_DIR}/{playbook}"
    cmd = ["ansible-playbook", "-i", INVENTORY, playbook_path]
    if limit_hosts:
        cmd.extend(["-l", ",".join(limit_hosts)])
    result = subprocess.run(cmd, capture_output=True, text=True)
    return result.returncode, result.stdout

def get_latest_connection_artifact():
    files = [f for f in os.listdir(ARTIFACTS_DIR) if f.startswith("connection_validator")]
    if not files:
        return None, None
    files.sort(reverse=True)
    latest = files[0]
    timestamp = latest.replace("connection_validator_", "").replace(".json", "")
    path = os.path.join(ARTIFACTS_DIR, latest)
    with open(path) as f:
        data = json.load(f)
    return data, timestamp

def load_check_server_name_artifacts():
    files = [f for f in os.listdir(ARTIFACTS_DIR) if f.startswith("check_server_name")]
    if not files:
        return None
    files.sort(reverse=True)
    latest = os.path.join(ARTIFACTS_DIR, files[0])
    with open(latest) as f:
        data = json.load(f)
    return data

def detect_user_intent(prompt):
    intent_prompt = f"""
You are an AI assistant for an Ansible automation system.
Determine what the user wants.
Possible intents:
FAILED_HOSTS_REPORT
RUN_HOSTNAME_CHECK
UNKNOWN
User request:
{prompt}
Respond with ONLY the intent name.
"""
    try:
        result = subprocess.run([OLLAMA_PATH, "run", OLLAMA_MODEL], input=intent_prompt, text=True, capture_output=True, timeout=OLLAMA_TIMEOUT)
        intent = result.stdout.strip()
        return intent
    except:
        return "UNKNOWN"

def analyze_host_with_llm(host):
    base_prompt = f"""
You are a senior infrastructure automation engineer.
Respond EXACTLY in this format:
Root Cause: <one sentence>
Suggested Fix: <actionable fix>
Host:
Hostname: {host.get('hostname')}
IP: {host.get('ip')}
Details: {host.get('details')}
"""
    try:
        result = subprocess.run([OLLAMA_PATH, "run", OLLAMA_MODEL], input=base_prompt, text=True, capture_output=True, timeout=OLLAMA_TIMEOUT)
        output = result.stdout.strip()
        root_cause = "Unknown"
        suggested_fix = "Manual investigation required"
        for line in output.splitlines():
            if line.lower().startswith("root cause"):
                root_cause = line.split(":",1)[1].strip()
            elif line.lower().startswith("suggested fix"):
                suggested_fix = line.split(":",1)[1].strip()
        return root_cause, suggested_fix
    except:
        return "LLM error", "Check Ollama service"

def generate_failed_hosts_csv(hosts, timestamp):
    rows = []
    with st.spinner("AI analyzing failed hosts..."):
        with concurrent.futures.ThreadPoolExecutor() as executor:
            future_map = {executor.submit(analyze_host_with_llm, host): host for host in hosts}
            for future in concurrent.futures.as_completed(future_map):
                host = future_map[future]
                root_cause, suggested_fix = future.result()
                rows.append([host.get("hostname"), host.get("ip"), timestamp, root_cause, suggested_fix])
    filename = f"{CSV_DIR}/failed_hosts_report_{datetime.now().strftime('%Y%m%d_%H%M%S')}.csv"
    with open(filename,"w",newline="") as f:
        writer = csv.writer(f)
        writer.writerow(["Hostname","IP Address","Last Playbook Run","Reason of Fail","Suggested Fix"])
        writer.writerows(rows)
    return filename, rows

def generate_hostname_comparison_csv(hosts):
    rows = [[host.get("hostname"), host.get("server_name")] for host in hosts]
    filename = f"{CSV_DIR}/hostname_comparison_{datetime.now().strftime('%Y%m%d_%H%M%S')}.csv"
    with open(filename,"w",newline="") as f:
        writer = csv.writer(f)
        writer.writerow(["Ansible Inventory Hostname","Real Hostname"])
        writer.writerows(rows)
    return filename, rows

if run_agent:
    if not user_prompt:
        st.warning("Please enter an instruction.")
        st.stop()
    intent = detect_user_intent(user_prompt)
    st.info(f"AI detected intent: {intent}")
    if intent == "FAILED_HOSTS_REPORT":
        data, timestamp = get_latest_connection_artifact()
        if data is None:
            st.error("No connection validator results found.")
            st.stop()
        failed_hosts = [h for h in data if not h.get("success")]
        if not failed_hosts:
            st.success("All hosts passed connectivity validation.")
            st.stop()
        filename, rows = generate_failed_hosts_csv(failed_hosts, timestamp)
        st.success("CSV report generated")
        st.dataframe(rows)
        with open(filename,"rb") as f:
            st.download_button("Download CSV",f,os.path.basename(filename))
    elif intent == "RUN_HOSTNAME_CHECK":
        st.info("Running hostname comparison playbook")
        rc, output = run_playbook("check_server_name.yml")
        st.code(output)
        if rc != 0:
            st.error("Playbook execution failed.")
            st.stop()
        data = load_check_server_name_artifacts()
        filename, rows = generate_hostname_comparison_csv(data)
        st.success("Hostname comparison report generated")
        st.dataframe(rows)
        with open(filename,"rb") as f:
            st.download_button("Download CSV",f,os.path.basename(filename))
    else:
        st.warning("AI could not determine the requested action.")
```

## Usage Workflow

### Run connection validator playbook

```
ansible-playbook -i ~/ansible/inventory.ini ~/ansible/connection_validator.yml
```

### Generate CSV report for failed hosts

* Open Streamlit
* Enter prompt: `Prepare CSV report of hosts that failed connection validator playbook`

### Run server name playbook on online hosts

* Enter prompt: `Run server name playbook on online hosts`

### Generate CSV report for successful server name checks

* Enter prompt: `Prepare CSV report of successful server name check playbook`

CSV includes actual server names obtained from hosts.
