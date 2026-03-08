# Ansible AI Agent Demo – Enhanced Version

This repository demonstrates an enhanced **Ansible AI Agent** using **Ansible**, **Streamlit**, and **Ollama LLM** to automate infrastructure analysis, playbook execution, and CSV reporting.

---

## Architecture Overview

| Component | Description |
|-----------|------------|
| **Ansible Controller VM** | Azure Linux VM (Ubuntu 24.04, Standard_E2s_v4, 2 vCPU, 16 GB RAM). Hosts Ansible, Python, Streamlit, and Ollama. |
| **Ansible Version** | Core 2.16.3 |
| **Python & venv** | Isolated Python virtual environment for dependencies |
| **Streamlit** | Version 1.54.0, provides web interface for user prompts and CSV reports |
| **LLM Model** | Ollama `phi3:mini` for AI analysis of failed hosts |
| **Managed Hosts** | 2 Azure VMs (Debian 12, Standard_D2s_v3), accessed via SSH by Ansible |

---

## Folder Structure

```text
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
artifacts/raw_runs contains standardized JSON outputs for both playbooks.

System Preparation

# Update system
sudo apt update && sudo apt upgrade -y

# Install Python and dependencies
sudo apt install -y python3 python3-venv python3-distutils python3-pip
python3 -m pip install --upgrade pip

# Install Ansible Core
python3 -m pip install --user ansible-core==2.16.3

# Setup SSH keys for managed hosts
ssh-keygen -t rsa -b 4096 -f ~/ansible-controller_key.pem
chmod 600 ~/ansible-controller_key.pem
ssh-copy-id -i ~/ansible-controller_key.pem pearlzfan@10.0.0.7
ssh-copy-id -i ~/ansible-controller_key.pem pearlzfan@10.0.0.8

Ansible Inventory (inventory.ini)

[linux]
host-vm1 ansible_host=10.0.0.7
host-vm2 ansible_host=10.0.0.8

[all:vars]
ansible_user=pearlzfan
ansible_ssh_private_key_file=/home/pearlzfan/ansible-controller_key.pem

Connection Validator Playbook (connection_validator.yml)

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


Server Name Playbook (check_server_name.yml)

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


Streamlit App (app.py)

```text
import streamlit as st
import json
import os
import subprocess
from datetime import datetime
import csv
import concurrent.futures

# =====================================================
# CONFIGURATION
# =====================================================
ARTIFACTS_DIR = "/home/pearlzfan/artifacts/raw_runs"
ANSIBLE_DIR = "/home/pearlzfan/ansible"
INVENTORY = f"{ANSIBLE_DIR}/inventory.ini"

OLLAMA_PATH = "/usr/local/bin/ollama"
OLLAMA_MODEL = "phi3:mini"
OLLAMA_TIMEOUT = 300

# =====================================================
# PRE-WARM OLLAMA MODEL
# =====================================================
def warm_up_model():
    try:
        subprocess.run(
            [OLLAMA_PATH, "run", OLLAMA_MODEL],
            input="Hello",
            text=True,
            capture_output=True,
            timeout=60
        )
    except:
        pass
warm_up_model()

# =====================================================
# STREAMLIT UI
# =====================================================
st.set_page_config(page_title="Ansible AI Agent", layout="wide")
st.title("Ansible AI Agent – Sequential Workflow")

user_prompt = st.text_area(
    "Enter instruction for the AI agent",
    placeholder="Example: Prepare CSV report of hosts that failed connection validator playbook"
)
run_agent = st.button("Run AI Agent")

# =====================================================
# HELPER FUNCTIONS
# =====================================================
def run_playbook(playbook, limit_hosts=None):
    playbook_path = f"{ANSIBLE_DIR}/{playbook}"
    cmd = ["ansible-playbook", "-i", INVENTORY, playbook_path]
    if limit_hosts:
        cmd.extend(["-l", ",".join(limit_hosts)])
    result = subprocess.run(cmd, capture_output=True, text=True)
    return result.returncode, result.stdout

def load_connection_validator_artifact():
    files = [f for f in os.listdir(ARTIFACTS_DIR) if f.startswith("connection_validator")]
    if not files:
        return None
    files.sort(reverse=True)
    latest = os.path.join(ARTIFACTS_DIR, files[0])
    with open(latest, "r") as f:
        return json.load(f)

def load_check_server_name_artifacts():
    files = [f for f in os.listdir(ARTIFACTS_DIR) if f.startswith("check_server_name")]
    if not files:
        return None
    combined = []
    for f in files:
        path = os.path.join(ARTIFACTS_DIR, f)
        with open(path, "r") as jf:
            data = json.load(jf)
            if isinstance(data, dict):
                combined.append(data)
            elif isinstance(data, list):
                combined.extend(data)
    return combined

# =====================================================
# LLM ANALYSIS FUNCTION
# =====================================================
def analyze_host_with_llm(host):
    base_prompt = f"""
You are a senior infrastructure automation engineer.
You must respond EXACTLY in this format:

Root Cause: <one concise sentence>
Suggested Fix: <clear, actionable remediation steps>

Do NOT add extra explanations or text.

Host Details:
Hostname: {host.get('hostname')}
IP: {host.get('ip')}
OS: {host.get('distribution', 'unknown')} ({host.get('os_family', 'unknown')})
Connectivity Success: {host.get('success')}
Details: {host.get('details')}
"""
    try:
        result = subprocess.run(
            [OLLAMA_PATH, "run", OLLAMA_MODEL],
            input=base_prompt,
            text=True,
            capture_output=True,
            timeout=OLLAMA_TIMEOUT
        )
        if result.returncode != 0:
            return f"Ollama error: {result.stderr.strip()}", "Model execution failed"
        output = result.stdout.strip()
        if not output:
            return "Empty LLM response", "Check Ollama service"
        root_cause = "Unknown"
        suggested_fix = "Manual investigation required"
        for line in output.splitlines():
            line = line.strip()
            if line.lower().startswith("root cause"):
                root_cause = line.split(":", 1)[1].strip()
            elif line.lower().startswith("suggested fix"):
                suggested_fix = line.split(":", 1)[1].strip()
        return root_cause, suggested_fix
    except subprocess.TimeoutExpired:
        return "LLM timeout", "Model took too long to respond"
    except Exception as e:
        return f"LLM exception: {str(e)}", "Check Streamlit logs"

# =====================================================
# CSV GENERATION FUNCTIONS
# =====================================================
def generate_llm_csv(hosts, status):
    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    rows = []
    with st.spinner("Analyzing hosts with AI..."):
        with concurrent.futures.ThreadPoolExecutor() as executor:
            future_to_host = {executor.submit(analyze_host_with_llm, host): host for host in hosts}
            for future in concurrent.futures.as_completed(future_to_host):
                host = future_to_host[future]
                root_cause, suggested_fix = future.result()
                rows.append([
                    host.get("hostname"),
                    host.get("ip"),
                    status,
                    timestamp,
                    root_cause,
                    suggested_fix
                ])
    filename = f"ai_report_{datetime.now().strftime('%Y%m%d_%H%M%S')}.csv"
    with open(filename, "w", newline="") as f:
        writer = csv.writer(f)
        writer.writerow([
            "Host",
            "IP Address",
            "Status",
            "Timestamp",
            "Root Cause",
            "Suggested Fix"
        ])
        writer.writerows(rows)
    return filename, rows

def generate_server_name_csv(hosts):
    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    rows = []
    for host in hosts:
        rows.append([
            host.get("hostname"),
            "Success" if host.get("success", False) else "Failed",
            timestamp,
            host.get("server_name", "unknown")
        ])
    filename = f"server_name_report_{datetime.now().strftime('%Y%m%d_%H%M%S')}.csv"
    with open(filename, "w", newline="") as f:
        writer = csv.writer(f)
        writer.writerow([
            "Hostname",
            "Playbook Execution Status",
            "Timestamp",
            "Server Name Obtained"
        ])
        writer.writerows(rows)
    return filename, rows

# =====================================================
# AI AGENT LOGIC
# =====================================================
if run_agent:
    if not user_prompt:
        st.warning("Please enter an instruction.")
        st.stop()
    prompt = user_prompt.lower()

    # Step 1: Failed connection CSV
    if "connection" in prompt and "failed" in prompt:
        st.info("Generating CSV report for failed hosts from connection validator")
        data = load_connection_validator_artifact()
        if data is None:
            st.error("No connection validator artifacts found.")
            st.stop()
        failed_hosts = [h for h in data if not h.get("success", False)]
        if not failed_hosts:
            st.success("All hosts passed connectivity validation.")
            st.stop()
        filename, rows = generate_llm_csv(failed_hosts, "Connection validation failed")
        st.success(f"CSV report generated: {filename}")
        st.dataframe(rows)
        with open(filename, "rb") as f:
            st.download_button("Download CSV", f, file_name=filename)

    # Step 2: Run server name playbook
    elif "run" in prompt and "server name" in prompt:
        st.info("Running check_server_name.yml playbook on online hosts only")
        data = load_connection_validator_artifact()
        if data is None:
            st.error("No connection validator artifacts found.")
            st.stop()
        online_hosts = [h for h in data if h.get("success", False)]
        if not online_hosts:
            st.warning("No hosts are online. Playbook will not run.")
            st.stop()
        hosts_str = [h["hostname"] for h in online_hosts]
        st.write("Hosts targeted:", hosts_str)
        rc, output = run_playbook("check_server_name.yml", limit_hosts=hosts_str)
        st.code(output)
        if rc == 0:
            st.success(f"Playbook executed successfully on {len(online_hosts)} hosts.")
        else:
            st.error("Playbook execution failed.")

    # Step 3: CSV of successful server name check
    elif "server name" in prompt and "success" in prompt:
        st.info("Generating CSV report for hosts where check_server_name succeeded")
        data = load_check_server_name_artifacts()
        if data is None:
            st.error("No server name artifacts found.")
            st.stop()
        successful_hosts = [h for h in data if h.get("success", False)]
        if not successful_hosts:
            st.warning("No successful executions found.")
            st.stop()
        filename, rows = generate_server_name_csv(successful_hosts)
        st.success(f"CSV report generated: {filename}")
        st.dataframe(rows)
        with open(filename, "rb") as f:
            st.download_button("Download CSV", f, file_name=filename)

    else:
        st.warning("Instruction not recognized. Use one of the three steps: generate failed connection CSV, run server name playbook, or generate successful server name CSV.")
```



Usage Workflow

Run connection validator playbook:

ansible-playbook -i ~/ansible/inventory.ini ~/ansible/connection_validator.yml

Generate CSV report for failed hosts:

Open Streamlit

Prompt: Prepare CSV report of hosts that failed connection validator playbook

Run server name playbook on online hosts:

Prompt: Run server name playbook on online hosts

Generate CSV report for successful server name checks:

Prompt: Prepare CSV report of successful server name check playbook

CSV includes actual server names obtained from hosts
