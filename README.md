# Ansible AI Agent Demo Architecture and Implementation

This document provides a detailed implementation plan for the Ansible + Streamlit + Ollama AI agent setup, including all commands, configurations, and final application code.

---

## 1. Architecture Overview

* **Ansible Controller VM:** Azure Linux VM, Ubuntu 24.04, Standard_E2s_v4 (2 vCPU, 16 GB RAM)
* **Ansible Version:** Core 2.16.3
* **Python & Virtual Environment:** Python 3 with `venv`
* **Streamlit:** 1.54.0 running in a Python virtual environment
* **LLM Model:** Ollama `phi3:mini`
* **Managed Hosts:** 2 VMs, Debian 12, Standard_D2s_v3

Ansible controller connects via SSH to the managed hosts to check connectivity using the `connection_validator.yml` playbook. Streamlit provides a web interface to prompt Ollama to generate CSV reports of failed hosts automatically.

---

## 2. Azure VM Setup

### 2.1. Ansible Controller VM

* Ubuntu 24.04
* Standard_E2s_v4 (2 vCPU, 16 GB RAM)

### 2.2. Managed Host VMs

* Debian 12
* Standard_D2s_v3

---

## 3. System Preparation Commands

### 3.1. Update and Upgrade OS

```bash
sudo apt update && sudo apt upgrade -y
```

### 3.2. Install Python3 and dependencies

```bash
sudo apt install -y python3 python3-venv python3-distutils python3-pip
python3 -m pip install --upgrade pip
```

### 3.3. Install Ansible Core

```bash
python3 -m pip install --user ansible-core==2.16.3
```

### 3.4. Set up SSH Keys for Managed Hosts

```bash
ssh-keygen -t rsa -b 4096 -f ~/ansible-controller_key.pem
chmod 600 ~/ansible-controller_key.pem
ssh-copy-id -i ~/ansible-controller_key.pem pearlzfan@10.0.0.7
ssh-copy-id -i ~/ansible-controller_key.pem pearlzfan@10.0.0.8
```

---

## 4. Ansible Inventory Configuration

Create file `~/ansible/inventory.ini`:

```ini
[linux]
host-vm1 ansible_host=10.0.0.7
host-vm2 ansible_host=10.0.0.8

[all:vars]
ansible_user=pearlzfan
ansible_ssh_private_key_file=/home/pearlzfan/ansible-controller_key.pem
```

---

## 5. Connection Validator Playbook

File: `~/ansible/connection_validator.yml`

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

    - name: Ping each host and append result
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
        all_results: "{{ all_results + [{
          'hostname': item.item,
          'default_ipv4': hostvars[item.item].ansible_host | default('unknown'),
          'os_family': hostvars[item.item].ansible_os_family | default('unknown'),
          'distribution': hostvars[item.item].ansible_distribution | default('unknown'),
          'connectivity_success': (item.ping is defined and item.ping == 'pong'),
          'connectivity_output': (item.ping if item.ping is defined and item.ping == 'pong' else 'Host unreachable or SSH failure')
        }] }}"
      loop: "{{ ping_result.results }}"
      loop_control:
        loop_var: item

    - name: Write JSON artifact
      copy:
        content: "{{ all_results | to_nice_json }}"
        dest: "{{ playbook_dir }}/../artifacts/raw_runs/connection_validator_{{ run_timestamp }}.json"
```

---

## 6. Python Virtual Environment & Streamlit

### 6.1. Create Virtual Environment

```bash
cd ~/ansible_web
python3 -m venv venv
source venv/bin/activate
```

### 6.2. Install Streamlit and Required Packages

```bash
pip install --upgrade pip
pip install streamlit==1.54.0 pandas
```

### 6.3. Run Streamlit Web App

```bash
cd ~/ansible_web
source venv/bin/activate
streamlit run app.py --server.port 8501 --server.address 0.0.0.0
```

Access the web interface via the external IP, e.g., `http://<controller_vm_ip>:8501`

---

## 7. Ollama Installation & Model

### 7.1. Install Ollama (assume already installed)

```bash
# Follow official Ollama instructions for Ubuntu
```

### 7.2. Pull phi3:mini Model

```bash
ollama pull phi3:mini
ollama list  # to confirm model is downloaded
```

---

## 8. Streamlit App (`app.py`)

```python
import streamlit as st
import json
import os
import subprocess
from datetime import datetime
import csv

# -------------------------
# CONFIG
# -------------------------
ARTIFACTS_DIR = "/home/pearlzfan/artifacts/raw_runs"  # path to your JSON files
OLLAMA_MODEL = "phi3:mini"  # small model for Streamlit
TIMEOUT_SEC = 30  # timeout for Ollama

# -------------------------
# STREAMLIT INTERFACE
# -------------------------
st.title("Ansible Connection Validator Report via Ollama")

st.markdown("""
This app allows you to prompt Ollama to generate a CSV report for hosts that failed the
connection validator playbook.
""")

# Prompt input
user_prompt = st.text_area("Prompt Ollama:", "Prepare CSV report of hosts that failed connection validator playbook.")

# -------------------------
# FIND JSON ARTIFACT
# -------------------------
json_files = sorted(
    [f for f in os.listdir(ARTIFACTS_DIR) if f.endswith(".json")],
    reverse=True
)

if not json_files:
    st.warning("No JSON artifacts found. Run connection_validator.yml first.")
    st.stop()

json_path = os.path.join(ARTIFACTS_DIR, json_files[0])
st.info(f"Using artifact: {json_files[0]}")

# -------------------------
# LOAD JSON
# -------------------------
with open(json_path, "r") as f:
    hosts_json = json.load(f)

# -------------------------
# FILTER FAILED HOSTS
# -------------------------
failed_hosts = [h for h in hosts_json if not h.get("connectivity_success", True)]

if not failed_hosts:
    st.success("All hosts passed connectivity. No failed hosts to report.")
    st.stop()

# -------------------------
# PREPARE INPUT FOR OLLAMA
# -------------------------
ollama_input = "\n".join(
    f"{h['hostname']} ({h['default_ipv4']}) - {h.get('error_message', '')}"
    for h in failed_hosts
)

final_prompt = f"{user_prompt}\n\n{ollama_input}"

# -------------------------
# GENERATE CSV BUTTON
# -------------------------
if st.button("Generate CSV via Ollama"):
    try:
        result = subprocess.run(
            ["ollama", "run", OLLAMA_MODEL],
            input=final_prompt.encode("utf-8"),
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            timeout=TIMEOUT_SEC
        )

        stdout = result.stdout.decode("utf-8").strip()
        stderr = result.stderr.decode("utf-8").strip()

        if result.returncode != 0:
            st.error(f"Ollama error:\n{stderr}")
        else:
            # Save CSV
            timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
            csv_filename = f"connection_validator_report_{timestamp}.csv"

            # Write CSV from Ollama output (assuming Ollama returns CSV-compatible text)
            csv_path = os.path.join(os.getcwd(), csv_filename)
            with open(csv_path, "w", newline="") as f:
                f.write(stdout)

            st.success(f"Report written to {csv_filename}")
            st.code(stdout)  # show CSV content

    except subprocess.TimeoutExpired:
        st.error(f"Ollama timed out after {TIMEOUT_SEC} seconds.")
    except Exception as e:
        st.error(f"Unexpected error: {e}")
```

---

## 9. Managed Hosts Setup (Debian 12)

No special setup is required beyond enabling SSH and using the Ansible inventory as defined. Ensure the controller can SSH using the private key.

---

## 10. Folder Structure

```
/home/pearlzfan/
├── ansible/
│   ├── connection_validator.yml
│   ├── inventory.ini
├── ansible_web/
│   ├── app.py
│   └── venv/
├── artifacts/
│   └── raw_runs/
```

---

## 11. Summary of Execution Steps

1. Run Ansible playbook:

```bash
ansible-playbook -i ~/ansible/inventory.ini ~/ansible/connection_validator.yml
```

2. Activate Python virtual environment and start Streamlit:

```bash
cd ~/ansible_web
source venv/bin/activate
streamlit run app.py --server.port 8501 --server.address 0.0.0.0
```

3. Open Streamlit URL in browser: `http://20.123.9.43:8501`
4. latest JSON artifact is selected automatically, enter prompt, e.g., "Prepare CSV report of hosts that failed connection validator playbook"
5. Click "Generate CSV" and download the report

This setup allows users to prompt Ollama from a web interface to automatically generate CSV reports based on Ansible connectivity results without manual file handling.
