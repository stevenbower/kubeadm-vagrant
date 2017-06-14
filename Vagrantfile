# -*- mode: ruby -*-
# vi: set ft=ruby :

# Install a kube cluster using kubeadm:
# http://kubernetes.io/docs/getting-started-guides/kubeadm/

# relevant environment variables:
# http_proxy, https_proxy, no_proxy: if set will be configured inside VMs
#   for proxy access
# cacerts_dir: directory on build host with extra CA certs to add
# worker_count: number of worker nodes to build (default 3)

require 'securerandom'

$generic_install_script = <<-EOH
  #!/bin/sh
  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
  echo 'deb http://apt.kubernetes.io/ kubernetes-xenial main' > /etc/apt/sources.list.d/kubernetes.list
  apt-get update
  apt-get install -y docker.io kubelet kubeadm kubectl kubernetes-cni
EOH

Vagrant.configure("2") do |config|
  kubeadm_token = "#{SecureRandom.hex[0...6]}.#{SecureRandom.hex[0...16]}"
  master_vm = 'master'
  master_vm_ip = '192.168.77.2'
  flannel_network = '10.244.0.0/16'
  update_ca_certificates_script = ''

  if Vagrant.has_plugin?("vagrant-proxyconf")
    # copy proxy settings from the Vagrant environment into the VMs
    http_proxy = ENV['http_proxy'] || ENV['HTTP_PROXY']
    https_proxy = ENV['https_proxy'] || ENV['HTTPS_PROXY']
    no_proxy = ENV['no_proxy'] || ENV['NO_PROXY']

    (no_proxy = [no_proxy, master_vm_ip].join(',')) unless no_proxy.nil?

    # vagrant-proxyconf will not configure anything if everything is nil
    config.proxy.http = http_proxy
    config.proxy.https = https_proxy
    config.proxy.no_proxy = no_proxy

    # look for another envvar that points to a directory with CA certs
    if ENV['cacerts_dir']
      cacerts_dir = Dir.new(ENV['cacerts_dir'])
      files_in_cacerts_dir = cacerts_dir.entries.select{ |e| not ['.', '..'].include? e }
      files_in_cacerts_dir.each do |f|
        next if File.directory?(File.join(cacerts_dir, f))
        begin
          unless f.end_with? '.crt'
            fail "All files in #{ENV['cacerts_dir']} must end in .crt due to update-ca-certificates restrictions."
          end
          # read in the certificate and normalize DOS line endings to UNIX
          cert_raw = File.read(File.join(ENV['cacerts_dir'], f)).gsub(/\r\n/, "\n")
          if cert_raw.scan('-----BEGIN CERTIFICATE-----').length > 1
            fail "Multiple certificates detected in #{File.join(ENV['cacerts_dir'], f)}, please split them into separate certificates."
          end
          cert = OpenSSL::X509::Certificate.new(cert_raw) # test that the cert is valid
          dest_cert_path = File.join('/usr/local/share/ca-certificates', f)
          update_ca_certificates_script << <<-EOH
            echo -ne "#{cert_raw}" > #{dest_cert_path}
          EOH
        rescue OpenSSL::X509::CertificateError
          fail "Certificate #{File.join(ENV['cacerts_dir'], f)} is not a valid PEM certificate, aborting."
        end # begin/rescue
      end # files_in_cacerts_dir.each
      update_ca_certificates_script << <<-EOH
        update-ca-certificates
      EOH
    end # if ENV['cacerts_dir']
  end # if Vagrant.has_plugin?

  config.vm.box = "ubuntu/xenial64"
  config.vm.box_check_update = false

  config.vm.provision 'configure-extra-certificates', type: 'shell' do |s|
    s.inline = update_ca_certificates_script
  end unless update_ca_certificates_script.empty?

  config.vm.provision 'generic-install', type: 'shell' do |s|
    s.inline = $generic_install_script
  end

  config.hostmanager.enabled = true
  config.hostmanager.manage_guest = true

  cpu_count = 1
  env_cpu_count = ENV['cpu_count'].to_i
  (cpu_count = env_cpu_count) unless env_cpu_count.zero?

  memory_mb = 1024
  env_memory_mb = ENV['memory_mb'].to_i
  (memory_mb = env_memory_mb) unless env_memory_mb.zero?

  config.vm.provider "virtualbox" do |vb|
    vb.cpus = cpu_count
    vb.memory = memory_mb
  end

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
        PROXY_CONFIG=/etc/profile.d/proxy.sh
        if [ -e $PROXY_CONFIG ]; then . $PROXY_CONFIG; fi
        curl -s -O https://raw.githubusercontent.com/rook/rook/master/demo/kubernetes/rook-operator.yaml
        curl -s -O https://raw.githubusercontent.com/rook/rook/master/demo/kubernetes/rook-cluster.yaml
        curl -s -O https://raw.githubusercontent.com/rook/rook/master/demo/kubernetes/rook-tools.yaml
      EOH
    end
  end

  worker_count = 3
  env_worker_count = ENV['worker_count'].to_i
  (worker_count = env_worker_count) unless env_worker_count.zero?

  worker_count.times do |i|
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
