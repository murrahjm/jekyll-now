# Ansible By Windows For Windows

* Ansible management tool overview
  * written in python so runs from a linux "control node"
  * playbooks run plays run tasks
  * declarative language but not as rigid as DSC
* Use Windows Subsystem for Linux as control node
  * install WSL
  * install ansible or compile from github source
  * other considerations
* manage windows clients from ansible playbooks
  * winrm connection requirements
  * credssp or constrained delegation
  * modules are written in powershell not python
  * dsc integration