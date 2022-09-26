.DEFAULT_GOAL := help

help:
	@echo How to running:
	@echo \# make install_helm
	@echo \# make install_kubecolor

install_helm:
	ansible-playbook -i inventory.yml main.yml -e install="helm"
install_kubecolor:
	ansible-playbook -i inventory.yml main.yml -e install="kubecolor"
install_kustomize:
	ansible-playbook -i inventory.yml main.yml -e install="kustomize"
