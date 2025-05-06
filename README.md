# Deployment of SPO data processing to VM

The project uses ansible to deploy a data processing pipeline for Smartphone Pressure Obsersvations (SPO) to a virtual machine (VM). The pipeline is designed to process and analyze data from the SPO project, which involves collecting pressure data from smartphones.

To manually deploy the pipeline, run the following command:

```bash
uv run ansible-playbook ansible/ewc.yml --inventory ansible/inventory --user <ewc_user> --private-key <private_key>
```
This command will execute the Ansible playbook defined in `playbook.yml` using the inventory file located in `inventory/hosts`. The playbook will set up the necessary environment and dependencies for the SPO data processing pipeline on the VM.
