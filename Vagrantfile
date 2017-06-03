# -*- mode: ruby -*-
# vi: set ft=ruby :

# Install a kube cluster using kubeadm:
# http://kubernetes.io/docs/getting-started-guides/kubeadm/

require 'securerandom'

$generic_install_script = <<-EOH
  #!/bin/sh
  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
  echo 'deb http://apt.kubernetes.io/ kubernetes-xenial main' > /etc/apt/sources.list.d/kubernetes.list
  apt-get update
  apt-get install -y docker.io kubelet kubeadm kubectl kubernetes-cni
EOH

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/xenial64"
  config.vm.box_check_update = false

  config.vm.provision 'generic-install', type: 'shell' do |s|
    s.inline = $generic_install_script
  end

  config.hostmanager.enabled = true
  config.hostmanager.manage_guest = true

  config.vm.provider "virtualbox" do |vb|
    vb.cpus = 1
    vb.memory = 1024
  end

  kubeadm_token = "#{SecureRandom.hex[0...6]}.#{SecureRandom.hex[0...16]}"
  master_vm = 'master'
  master_vm_ip = '192.168.77.2'
  flannel_network = '10.244.0.0/16'

  config.vm.define master_vm do |c|
    c.vm.hostname = master_vm
    c.vm.network 'private_network', ip: master_vm_ip

    c.vm.provision 'kubeadm-init', type: 'shell' do |s|
      s.inline = "sudo kubeadm init --apiserver-advertise-address #{master_vm_ip} --pod-network-cidr #{flannel_network} --token #{kubeadm_token}"
    end

    c.vm.provision 'copy-kubeadm-config', type: 'shell' do |s|
      s.privileged = false
      s.inline = <<-EOH
        #!/bin/sh
        mkdir -p $HOME/.kube
        sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config
      EOH
    end

    c.vm.provision 'install-flannel', type: 'shell' do |s|
      s.privileged = false
      s.inline = <<-EOH
        #!/bin/sh
        curl -s -O https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel-rbac.yml
        curl -s -O https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
        export FLANNEL_IFACE=$(ip a | grep #{master_vm_ip} | awk '{ print $NF }')
        sed -r -i -e "s|command: \\[ \\"/opt/bin/flanneld\\", \\"--ip-masq\\", \\"--kube-subnet-mgr\\" \\]|command: \\[ \\"/opt/bin/flanneld\\", \\"--ip-masq\\", \\"--kube-subnet-mgr\\", \\"--iface\\", \\"${FLANNEL_IFACE}\\" \\]|" kube-flannel.yml
        kubectl create -f $HOME/kube-flannel-rbac.yml
        sleep 2
        kubectl create -f $HOME/kube-flannel.yml
      EOH
    end
    
    c.vm.provision 'pull-rook-manifests', type: 'shell' do |s|
      s.privileged = false
      s.inline = <<-EOH
        curl -s -O https://raw.githubusercontent.com/rook/rook/master/demo/kubernetes/rook-operator.yaml
        curl -s -O https://raw.githubusercontent.com/rook/rook/master/demo/kubernetes/rook-cluster.yaml
        curl -s -O https://raw.githubusercontent.com/rook/rook/master/demo/kubernetes/rook-tools.yaml
      EOH
    end
  end

  3.times do |i|
    idx = i+1
  
    vm_name = "w#{idx}"
    vm_ip = (master_vm_ip.split('.')[0..2] + [10+idx]).join('.')
    config.vm.define vm_name do |c|
      c.vm.hostname = vm_name
      c.vm.network "private_network", ip: vm_ip
      
      c.vm.provision 'join-kubernetes-cluster', type: 'shell' do |s|
        s.inline = "sudo kubeadm join --token #{kubeadm_token} #{master_vm_ip}:6443"
      end
    end
  end
end
