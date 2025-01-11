# Simplon_Ansible

Automatisation de la gestion des systèmes avec Ansible

L'utilisation d'Ansible pour automatiser la configuration et le durcissement de deux cibles distinctes : une machine Linux et une machine Windows Server 2022. Deux fichiers Playbook spécifiques ont été créés et exécutés pour accomplir ces tâches.

### 1\. **Configuration des hôtes (fichier `hosts`)**

Le fichier `hosts` définit les cibles et leurs paramètres de connexion. Il contient deux groupes principaux :

-   **Linux** : Une machine Debian avec les détails de connexion SSH.
-   **Windows** : Un serveur Windows 2022 configuré pour être accessible via WinRM avec Kerberos.

```
targets:
  hosts:
    linux:
      ansible_host: 10.0.0.7
      ansible_user: cyril
      ansible_password: 
    windows:
      ansible_host: WIN-AD
      ansible_user: administrator
      ansible_password: 
      ansible_connection: winrm
      ansible_port: 5985
      ansible_winrm_transport: kerberos

```

### 2\. **Playbook Linux (`playbook-linux.yml`)**

Le Playbook pour Linux effectue des tâches de maintenance et de durcissement du système, notamment :

-   Mise à jour des paquets.
-   Installation et configuration de pare-feu (UFW).
-   Désactivation de la connexion root via SSH.
-   Configuration de SELinux.
-   Ajout d'un délai d'inactivité pour SSH.

Syntaxe des playbooks : https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_intro.html#playbook-syntax


```
---
- name: Update system packages
  hosts: linux
  become: yes
  #become_user: cyril
  tasks:
    - name: Update package cache
      apt:
        update_cache: yes
        state: present

    - name: Upgrade system packages
      apt:
        upgrade: yes
        state: latest

    - name: Ensure UFW is installed
      apt:
        name: ufw
        state: present

    - name: Disable root login
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^PermitRootLogin'
        line: 'PermitRootLogin no'
        create: yes

    - name: Ensure SELinux is installed and enabled
      apt:
        name: selinux-basics
        state: present

    - name: Apply SELinux configuration
      command: selinux-activate
      args:
        creates: /etc/selinux/config

    - name: Set SSH idle timeout interval
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^ClientAliveInterval'
        line: 'ClientAliveInterval 300'
      notify: Restart SSH

```

### 3\. **Playbook Windows (`playbook-windows.yml`)**

Le Playbook pour Windows se concentre sur le durcissement de Windows Server 2022 et inclut :

-   Application des mises à jour critiques et de sécurité.
-   Configuration de pare-feu pour autoriser le RDP.
-   Désactivation de SMBv1 et activation de SMBv2/SMBv3.
-   Activation des audits pour la gestion des comptes et des connexions.
-   Configuration de Windows Defender pour des mises à jour quotidiennes.

```
---
- name: Windows Server 2022 Hardening
  hosts: windows
  become: yes
  tasks:

    - name: Ensure Windows is up to date
      win_updates:
        category_names:
          - CriticalUpdates
          - SecurityUpdates
        reboot: yes

    - name: Allow RDP through the firewall
      win_firewall_rule:
        name: "AllowRDP"
        localport: 3389
        protocol: TCP
        action: allow
        direction: in
        enabled: yes

    - name: Disable SMBv1
      win_feature:
        name: FS-SMB1
        state: absent

    - name: Enable SMBv2 and SMBv3
      win_feature:
        name: FS-SMB2
        state: present

    - name: Enable auditing of logon events
      win_audit_policy:
        name: "Logon/Logoff"
        subcategory: "Logon"
        audit_flag: Success,Failure

    - name: Enable auditing of account management
      win_audit_policy:
        name: "Account Management"
        subcategory: "User Account Management"
        audit_flag: Success,Failure

    - name: Ensure Windows Defender Antivirus is enabled
      win_feature:
        name: Windows-Defender-Features
        state: present

    - name: Configure Windows Defender Antivirus to update daily
      win_scheduled_task:
        name: "DefenderUpdate"
        description: "Update Windows Defender Antivirus daily"
        actions:
          - path: "%ProgramFiles%/Windows Defender/MpCmdRun.exe"
            arguments: "-SignatureUpdate"
        triggers:
          - type: daily
            start_boundary: "2023-01-01T02:00:00"
        state: present

    - name: Enable Windows Update automatic updates
      win_service:
        name: wuauserv
        start_mode: auto
        state: started

    - name: Disable LM and NTLMv1
      win_regedit:
        path: HKLM:\\SYSTEM\\CurrentControlSet\\Control\\Lsa
        name: "LmCompatibilityLevel"
        data: 5
        type: dword

    - name: Ensure PowerShell script execution policy is restricted
      win_shell: Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope LocalMachine

```

### 4\. **Exécution des Playbooks**

Pour exécuter les playbooks, la commande suivante est utilisée :

Pour Linux :

```
ansible-playbook -i hosts playbook-linux.yml

```

Pour Windows :

```
ansible-playbook -i hosts playbook-windows.yml

```

Ces commandes ciblent les machines spécifiées dans le fichier `hosts` et appliquent les configurations définies dans les playbooks.

### **Vérification de l'exécution**

Pour s'assurer que les tâches ont été correctement exécutées :

-   Sur Linux, utilisez la commande `ufw status` pour vérifier le pare-feu ou consultez `/etc/ssh/sshd_config` pour valider les modifications SSH.
-   Sur Windows, vérifiez les paramètres de pare-feu via `Get-NetFirewallRule` ou consultez les journaux d'audit pour valider les activations.

* * * * *

### Conclusion

L'utilisation d'Ansible a permis d'automatiser efficacement le durcissement et la maintenance des systèmes Linux et Windows Server. Cela garantit une conformité renforcée tout en réduisant le temps manuel nécessaire pour appliquer ces configurations sur plusieurs systèmes.