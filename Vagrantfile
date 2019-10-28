# -*- mode: ruby -*-
# # vi: set ft=ruby :

# For help on using kubespray with vagrant, check out docs/vagrant.md

require 'fileutils'

Vagrant.require_version ">= 2.0.0"

# Uniq disk UUID for libvirt
DISK_UUID = Time.now.utc.to_i

SUPPORTED_OS = {
  "centos"      => { user: "vagrant", box: "centos/7"},
  "ubuntu1604"  => { user: "vagrant", box: "generic/ubuntu1604"},
  "ubuntu1804"  => { user: "vagrant", box: "generic/ubuntu1804"},
}

host_vars = {}

######################################################################
# Local Kubernetes Cluster on dynamic number of nodes.
######################################################################

# Require YAML module
require 'yaml'

# Read YAML file with infrastructure details
hosts = YAML.load_file('k8s-nodes.yml')

# ~~ General Variables
host_vars = {}

# Total Number of Nodes
$total=0
$count=0

hosts["node_types"].each do | server_type, settings |
  $total += settings["number"]
end

######################################################################
# Kubespray settings
######################################################################


# Defaults for config options defined in CONFIG
$num_instances   = $total
$vm_gui          = false
$vm_memory       = 2048
$vm_cpus         = 1
$shared_folders  = {}
$forwarded_ports = {}
$subnet          = "10.0.20"
$os              = "centos"
$network_plugin  = "flannel"

# Setting multi_networking to true will install Multus:
# https://github.com/intel/multus-cni
$multi_networking = false

# The first three nodes are etcd servers
$etcd_instances = 1
# The first two nodes are kube masters
$kube_master_instances = 1

# All nodes are kube nodes
$kube_node_instances = $num_instances
# The following only works when using the libvirt provider
$kube_node_instances_with_disks = false
$kube_node_instances_with_disks_size = "20G"
$kube_node_instances_with_disks_number = 2
$override_disk_size = false
$disk_size = "20GB"
$local_path_provisioner_enabled = false
$local_path_provisioner_claim_root = "/opt/local-path-provisioner/"

$playbook = "kubespray/cluster.yml"
$box = SUPPORTED_OS[$os][:box]

$inventory = File.dirname(__FILE__)

######################################################################
# Show time
######################################################################

nodes = Array.new
masters = Array.new

Vagrant.configure("2") do |config|

  config.vm.box = $box
  config.vm.box_check_update = false
  config.ssh.username = "vagrant"

  config.vm.provider 'virtualbox' do |box|
    box.customize [ "modifyvm", :id, "--macaddress1", "auto" ]
    box.customize [ "modifyvm", :id, "--nestedpaging", "on" ]
    box.customize [ "modifyvm", :id, "--ioapic", "on" ]
    box.customize [ "modifyvm", :id, '--audio', 'none']
    box.customize [ "modifyvm", :id, "--usbxhci", "off", "--usb", "off", "--usbehci", "off" ]
    box.customize [ "modifyvm", :id, "--vram", "8"]
    box.linked_clone = true
  end



  hosts["node_types"].each do | server_type, settings |
    (1..settings["number"]).each do |id|
      config.vm.define "#{server_type}-#{id}" do |box|
        $count += 1

        $name = "#{server_type}-#{id}"
        $ip   = "#{$subnet}.#{100+$count}"

        if $name.start_with?("node-")
          nodes << $name
        end


        if $name.start_with?("master-")
          masters << $name
        end


        # ~* Node Specific Settings *~
        box.vm.provider 'virtualbox' do |cfg|
          cfg.memory = settings["ram"]
          cfg.customize [ "modifyvm", :id, "--memory", settings["ram"] ]
          cfg.cpus = settings["cpu"]
          cfg.customize [ "modifyvm", :id, "--cpus",   settings["cpu"] ]
          cfg.customize [ "modifyvm", :id, "--cpuexecutioncap", "100", ]
          cfg.customize [ "modifyvm", :id, "--vram",   "8"]
        end

        config.vagrant.plugins = [
          "vagrant-sshfs",
          "vagrant-persistent-storage"
        ]

        config.vm.synced_folder ".", "/vagrant",
          disabled: false,
          type: "rsync",
          rsync__args: ['--verbose', '--archive', '--delete', '-z'],
          rsync__exclude: ['.git','venv']

        $shared_folders.each do |src, dst|
          config.vm.synced_folder src, dst,
            type: "rsync",
            rsync__args: ['--verbose', '--archive', '--delete', '-z']
        end

        # print $name, " ", $ip, "\n"

        # ~* Hostname and IP *~
        box.vm.hostname = $name
        box.vm.network "private_network", ip: $ip

        box.persistent_storage.location     = "volumes/#{$name}.vdi"
        box.persistent_storage.enabled      = true
        box.persistent_storage.size         = 20460
        box.persistent_storage.partition    = false
        box.persistent_storage.variant      = "Standard"
        box.persistent_storage.use_lvm      = false
        box.persistent_storage.mountname    = 'nfs-provisioner'
        box.persistent_storage.filesystem   = 'ext4'
        box.persistent_storage.mountpoint   = '/srv/nfs-provisioner'


        host_vars[$name] = {
          "ip": $ip,
          "flannel_interface": "eth1",
          "kube_network_plugin": $network_plugin,
          "kube_network_plugin_multus": $multi_networking,
          "download_run_once":  "True",
          "download_localhost": "False",
          "download_cache_dir": "cache",

          # Make kubespray cache even when download_run_once is false
          "download_force_cache": "True",

          # Keeping the cache on the nodes can improve provisioning speed while debugging kubespray
          "download_keep_remote_cache": "False",
          "docker_keepcache": "1",

          # These two settings will put kubectl and admin.config in $inventory/artifacts
          # kubeconfig
          "kubeconfig_localhost": "True",

          # kubectl
          "kubectl_localhost": "False",
          "local_path_provisioner_enabled": "#{$local_path_provisioner_enabled}",
          "local_path_provisioner_claim_root": "#{$local_path_provisioner_claim_root}",
          "ansible_ssh_user": SUPPORTED_OS[$os][:user],
        }


        if $count == $total
          File.open('.hosts.yml', 'w') { |f| f.write host_vars.to_yaml }

          box.vm.provision "ansible" do |ansible|
            ansible.playbook = $playbook
            $ansible_inventory_path = File.join( $inventory, "/hosts")

            ansible.become = true
            ansible.limit = "all,localhost"
            ansible.host_key_checking = false
            ansible.raw_arguments = ["--forks=#{$num_instances}", "--flush-cache", "-e ansible_become_pass=vagrant"]
            ansible.host_vars = host_vars
            ansible.groups = {
              "etcd" => masters,
              "kube-master" => masters,
              "kube-node" => nodes,
              "k8s-cluster:children" => [ "kube-master", "kube-node" ],
            }
          end

        end


        # Disable swap for each vm
        box.vm.provision "shell", inline: "swapoff -a"
      end
    end
  end

  # always use Vagrants insecure key
  config.ssh.insert_key = false
end
