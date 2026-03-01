# Ansible AI Agent Demo Architecture and Implementation

This document provides a detailed implementation plan for the Ansible + Streamlit + Ollama AI agent setup, including all commands, configurations, and final application code.

---

## 1. Architecture Overview

* **Ansible Controller VM:** Azure Linux VM, Ubuntu 24.04, Standard_E2s_v4 (2 vCPU, 16 GB RAM)
* **Ansible Version:** Core 2.16.3
* **Python & Virtual Environment:** Python 3 with `venv`
* **Streamlit:** 1.54.0 running in a Python virtual environment
* **LLM Model:** Ollama `phi3:mini`
* **Managed Hosts (Simulated):** 2 VMs, Debian 12, Standard_D2s_v3

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
- name: Connection Validator — real hosts
  hosts: linux
  gather_facts: yes
  tasks:
    - name: Initialize results list on localhost
      set_fact:
        connectivity_results: []
      delegate_to: localhost

    - name: Append host info to results
      set_fact:
        connectivity_results: "{{ connectivity_results + [ {
          'hostname': inventory_hostname,
          'default_ipv4': ansible_default_ipv4.address | default('unknown'),
          'distribution': ansible_distribution | default('unknown'),
          'os_family': ansible_os_family | default('unknown'),
          'connectivity_success': (ansible_facts is defined) }} ] }}"
      delegate_to: localhost

    - name: Write JSON artifact
      copy:
        content: "{{ connectivity_results | to_nice_json }}"
        dest: "/home/pearlzfan/artifacts/raw_runs/connection_validator_{{ ansible_date_time.iso8601_basic }}.json"
      delegate_to: localhost
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
import pandas as pd
import subprocess

ARTIFACTS_DIR = "/home/pearlzfan/artifacts/raw_runs"
MODEL_NAME = "phi3:mini"

st.title("Ansible Connection Validator AI Report")

# List JSON artifacts
json_files = sorted([f for f in os.listdir(ARTIFACTS_DIR) if f.endswith('.json')], reverse=True)

if not json_files:
    st.warning("No JSON artifacts found. Run connection_validator.yml first.")
else:
    selected_file = st.selectbox("Select JSON artifact", json_files)

    user_prompt = st.text_area("Prompt for Ollama", "Prepare CSV report of hosts that failed connection validator playbook")

    if st.button("Generate CSV"):
        artifact_path = os.path.join(ARTIFACTS_DIR, selected_file)
        with open(artifact_path, 'r') as f:
            json_data = f.read()

        full_prompt = f"{user_prompt}\nHere is the JSON data:\n{json_data}"

        try:
            result = subprocess.run(
                ["ollama", "run", MODEL_NAME],
                input=full_prompt.encode('utf-8'),
                capture_output=True,
                check=True
            )
            csv_output = result.stdout.decode('utf-8')

            # Save CSV
            csv_file = f"connection_validator_report_{selected_file.replace('.json','.csv')}"
            with open(csv_file, 'w') as f:
                f.write(csv_output)

            st.success(f"Report written to {csv_file}")
            st.download_button("Download CSV", csv_output, file_name=csv_file, mime='text/csv')

        except subprocess.CalledProcessError as e:
            st.error(f"Ollama error: {e.stderr.decode('utf-8')}")
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

3. Open Streamlit URL in browser: `http://<controller_ip>:8501`
4. Select JSON artifact and enter prompt, e.g., "Prepare CSV report of hosts that failed connection validator playbook"
5. Click "Generate CSV" and download the report

This setup allows users to prompt Ollama from a web interface to automatically generate CSV reports based on Ansible connectivity results without manual file handling.
