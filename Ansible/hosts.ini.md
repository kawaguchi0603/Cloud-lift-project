[nginx_lb]
100.97.100.23  ansible_user=keita_kawaguchi  ansible_ssh_private_key_file=~/.ssh/id_rsa  ansible_become=yes  ansible_become_method=sudo  ansible_become_password=""

[gcp_mysql_master]
100.90.208.68  ansible_user=keita_kawaguchi  ansible_ssh_private_key_file=~/.ssh/id_rsa  ansible_become=yes  ansible_become_method=sudo  ansible_become_password=""

[gcp_mysql_slave]
100.115.57.3  ansible_user=keita_kawaguchi  ansible_ssh_private_key_file=~/.ssh/id_rsa  ansible_become=yes  ansible_become_method=sudo  ansible_become_password=""
