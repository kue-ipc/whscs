# PythonとAnsible

## 参考資料

PythonとAnsibleのバージョン

<https://docs.ansible.com/projects/ansible/latest/reference_appendices/release_and_maintenance.html>

RHEL系における利用可能なPythonのバージョン

<https://access.redhat.com/support/policy/updates/rhel-app-streams-life-cycle>

RHEL8では3.6であるため、RHEL8に対応する場合は2.16以下にする必要がある。

```bash
pip install --user 'ansible-core<2.17' ansible
```
