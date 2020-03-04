# -*- mode: ruby -*-
# vi: set ft=ruby :

require "yaml"
hosts = YAML.load_file('vagrant_hosts.yml')

VAGRANTFILE_API_VERSION = '2'

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  # Global settings
  config.ssh.insert_key = false
  config.vm.box_check_update = false
  config.vbguest.auto_update = false
  config.vm.box = hosts['box']

  hosts['nodes'].each do |host|
    config.vm.define host['name'] do |node|
      node.vm.hostname = host['name']
      node.vm.network hosts['net']['network_type'], ip: host['ip']

      node.vm.provider :virtualbox do |vb|
        vb.customize [
          "modifyvm", :id,
          "--memory", host["memory"]
        ]
        vb.customize [
          "modifyvm", :id,
          "--cpus", host["cpus"]
        ]
      end
    end
  end

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "provisioning/initialize_new_droplet.yml"
    ansible.become = true
  end

end
