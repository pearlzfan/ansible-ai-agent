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

# =====================================================
# CONFIGURATION
# =====================================================

ARTIFACTS_DIR = "/home/pearlzfan/artifacts/raw_runs"
OLLAMA_MODEL = "phi3:mini"
OLLAMA_TIMEOUT = 120

# =====================================================
# PRE-WARM OLLAMA MODEL
# =====================================================

try:
    subprocess.run(
        ["ollama", "pull", OLLAMA_MODEL],
        stdout=subprocess.DEVNULL,
        stderr=subprocess.DEVNULL
    )
except Exception:
    pass

# =====================================================
# STREAMLIT UI
# =====================================================

st.set_page_config(page_title="Ansible AI Agent", layout="wide")
st.title("Ansible AI Agent – Connection Failure Analysis")

# =====================================================
# CUSTOM USER PROMPT FIELD
# =====================================================

user_prompt = st.text_area(
    "Optional: Add custom instructions for AI analysis",
    placeholder="Example: Provide concise enterprise-grade remediation steps."
)

# =====================================================
# LOAD LATEST JSON ARTIFACT
# =====================================================

if not os.path.exists(ARTIFACTS_DIR):
    st.error("Artifacts directory not found.")
    st.stop()

json_files = sorted(
    [f for f in os.listdir(ARTIFACTS_DIR) if f.endswith(".json")],
    reverse=True
)

if not json_files:
    st.warning("No JSON artifacts found. Run connection_validator.yml first.")
    st.stop()

latest_file = json_files[0]
json_path = os.path.join(ARTIFACTS_DIR, latest_file)

st.info(f"Using artifact: {latest_file}")

with open(json_path, "r") as f:
    hosts_data = json.load(f)

# =====================================================
# FILTER FAILED HOSTS
# =====================================================

failed_hosts = [
    h for h in hosts_data
    if not h.get("connectivity_success", True)
]

if not failed_hosts:
    st.success("All hosts passed connectivity validation.")
    st.stop()

st.write(f"Detected {len(failed_hosts)} failed host(s).")

# =====================================================
# LLM ANALYSIS FUNCTION (IMPROVED)
# =====================================================

def analyze_host_with_llm(host, custom_instruction):

    base_instruction = """
You are a senior infrastructure automation engineer.

You MUST respond EXACTLY in this format:

Root Cause: <one short sentence>
Suggested Fix: <clear remediation steps>

Do NOT add explanations before or after.
"""

    instruction_block = (
        f"\nAdditional User Instructions:\n{custom_instruction}\n"
        if custom_instruction.strip()
        else ""
    )

    prompt = f"""
{base_instruction}
{instruction_block}

Host: {host.get('hostname')}
IP: {host.get('default_ipv4')}
Distribution: {host.get('distribution', 'unknown')}
OS Family: {host.get('os_family', 'unknown')}
Connectivity Success: {host.get('connectivity_success')}
"""

    try:
        result = subprocess.run(
            ["ollama", "run", OLLAMA_MODEL],
            input=prompt,
            text=True,
            capture_output=True,
            timeout=OLLAMA_TIMEOUT
        )

        if result.returncode != 0:
            return (
                "LLM execution error",
                result.stderr.strip() or "Model returned non-zero exit code"
            )

        output = result.stdout.strip()

        if not output:
            return (
                "Empty LLM response",
                "Model returned no content"
            )

        # -------------------------------
        # STRICT FORMAT PARSING
        # -------------------------------
        root_cause = None
        suggested_fix = None

        for line in output.splitlines():
            line = line.strip()

            if line.lower().startswith("root cause"):
                root_cause = line.split(":", 1)[1].strip()

            elif line.lower().startswith("suggested fix"):
                suggested_fix = line.split(":", 1)[1].strip()

        # -------------------------------
        # FALLBACK SMART PARSING
        # -------------------------------
        if not root_cause or not suggested_fix:

            lines = [l.strip() for l in output.splitlines() if l.strip()]

            if len(lines) >= 2:
                root_cause = lines[0]
                suggested_fix = lines[1]
            else:
                root_cause = output[:200]
                suggested_fix = "Manual review required"

        return root_cause, suggested_fix

    except subprocess.TimeoutExpired:
        return (
            "Model response timeout",
            "Increase timeout or verify Ollama performance"
        )

    except Exception as e:
        return (
            "Unexpected LLM error",
            str(e)
        )

# =====================================================
# GENERATE CSV REPORT
# =====================================================

if st.button("Generate AI CSV Report"):

    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    report_rows = []

    with st.spinner("Analyzing failures with AI model..."):

        for host in failed_hosts:

            root_cause, suggested_fix = analyze_host_with_llm(
                host,
                user_prompt
            )

            report_rows.append([
                host.get("hostname"),
                host.get("default_ipv4"),
                "Connection Validation Failed",
                timestamp,
                root_cause,
                suggested_fix
            ])

    csv_filename = f"ai_connection_report_{datetime.now().strftime('%Y%m%d_%H%M%S')}.csv"
    csv_path = os.path.join(os.getcwd(), csv_filename)

    with open(csv_path, "w", newline="") as f:
        writer = csv.writer(f)
        writer.writerow([
            "Host",
            "IP Address",
            "Reason for Failure",
            "Time Detected",
            "Root Cause Analysis",
            "Suggested Fix"
        ])
        writer.writerows(report_rows)

    st.success(f"Report generated: {csv_filename}")

    st.dataframe(report_rows)

    with open(csv_path, "rb") as f:
        st.download_button(
            label="Download CSV Report",
            data=f,
            file_name=csv_filename,
            mime="text/csv"
        )
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
