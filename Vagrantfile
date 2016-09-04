# -*- mode: ruby -*-
# vi: set ft=ruby :


nodes = {
  'node1' => {
    'ip' => '192.168.70.101',
    'groups' => ['mysql_nodes', 'cluster_nodes'],
    'host_vars' => {
      "mysql_server_id" => "1",
      "mysql_replication_role" => 'master',
    },
  },
  'node2' => {
    'ip' => '192.168.70.102',
    'groups' => ['mysql_nodes', 'cluster_nodes'],
    'host_vars' => {
      "mysql_server_id" => "2",
      "mysql_replication_role" => 'slave'
    },
  },
  'node3' => {
    'ip' => '192.168.70.103',
    'groups' => ['cluster_nodes'],
  }
}

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = 'minimal/centos6'
  config.vm.boot_timeout = 600

  config.hostmanager.enabled = true

  config.vm.provider :virtualbox do |vb|
    vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
    vb.customize ["modifyvm", :id, "--memory", 1024]
  end

  # Create all nodes according to their spec
  nodes.each_pair { |node, node_params|
    config.vm.define node do |node_config|
      node_config.vm.hostname = node
      node_config.vm.network :private_network, ip: node_params['ip']
    end
  }

  config.vm.provision :ansible do |ansible|
    ansible.verbose = 'vv'
    ansible.sudo = true
    ansible.groups = Hash.new
    ansible.host_vars = Hash.new
    nodes.each_pair { |node, node_config|
      node_config['groups'].each { |group|
        if ansible.groups.include?(group)
          ansible.groups[group].push(node)
        else
          ansible.groups[group] = [node]
        end
      }
      ansible.host_vars[node] = node_config['host_vars']
    }
    ansible.extra_vars = {
        ansible_ssh_user: 'vagrant',
    }
    # Disable default limit (required with Vagrant 1.5+)
    ansible.limit = 'all'

    ansible.playbook = "provisioning/site.yml"
    ansible.host_key_checking = false
  end
end

