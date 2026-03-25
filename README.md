# lab-devspaces-ansible-exercise2

chmod 600 ssh_tests_connections/id_fedora_new

ansible-galaxy collection install community.general

ansible-playbook -i inventory deploy-wildfly.yaml

ssh user1@10.234.2.26 -i ssh_tests_connections/id_fedora_new

curl localhost:8080/sample/

ansible-lint *

yamllint *