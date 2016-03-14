# -*- mode: ruby -*-
# vi: set ft=ruby :

# This Vagrantfile helps test the devtest role on base centos and ubuntu vms

base_ip = "192.168.2."

num_nodes = 2
if ENV['CONTIV_NODES'] && ENV['CONTIV_NODES'] != "" then
    num_nodes = ENV['CONTIV_NODES'].to_i
end
node_ips = num_nodes.times.collect { |n| base_ip + "#{n+10}" }
node_names = num_nodes.times.collect { |n| "cluster-node#{n+1}" }

host_env = { }
if ENV['CONTIV_ENV'] then
    ENV['CONTIV_ENV'].split(" ").each do |env|
        e = env.split("=")
        host_env[e[0]]=e[1]
    end
end

if ENV["http_proxy"]
  host_env["HTTP_PROXY"]  = host_env["http_proxy"]  = ENV["http_proxy"]
  host_env["HTTPS_PROXY"] = host_env["https_proxy"] = ENV["https_proxy"]
  host_env["NO_PROXY"]    = host_env["no_proxy"]    = ENV["no_proxy"]
end

ansible_groups = { }
ansible_playbook = ENV["CONTIV_ANSIBLE_PLAYBOOK"] || "./site.yml"
ansible_tags =  ENV["CONTIV_ANSIBLE_TAGS"] || "prebake-for-dev"
ansible_extra_vars = {
    "env" => host_env,
    "service_vip" => node_ips[0],
    "validate_certs" => "no",
    "control_interface" => "eth1",
    "netplugin_if" => "eth2",
    "docker_version" => "1.10.2",
    "etcd_peers_group" => "netplugin-node",
    "scheduler_provider" => "none"
}

Vagrant.configure(2) do |config|
    (0..num_nodes-1).each do |n|
        node_name = node_names[n]
        node_addr = node_ips[n]
        config.vm.define node_name do |node|
            node.vm.hostname = node_name
            node.vm.box = "puppetlabs/ubuntu-14.04-64-nocm"
            node.vm.box_version = "1.0.3"
            # create an interface for cluster (control) traffic
            node.vm.network :private_network, ip: node_addr, virtualbox__intnet: "true"
            # create an interface for cluster (data) traffic
            node.vm.network :private_network, ip: "0.0.0.0", virtualbox__intnet: "true"
            node.vm.provider "virtualbox" do |v|
                # give enough ram and memory for docker to run fine
                v.customize ['modifyvm', :id, '--memory', "4096"]
                v.customize ["modifyvm", :id, "--cpus", "2"]
                # make all nics 'virtio' to take benefit of builtin vlan tag
                # support, which otherwise needs to be enabled in Intel drivers,
                # which are used by default by virtualbox
                v.customize ['modifyvm', :id, '--nictype1', 'virtio']
                v.customize ['modifyvm', :id, '--nictype2', 'virtio']
                v.customize ['modifyvm', :id, '--nictype3', 'virtio']
                v.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
                v.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all']
            end

            if ansible_groups["netplugin-node"] == nil then
                ansible_groups["netplugin-node"] = [ ]
            end
            ansible_groups["netplugin-node"] << node_name
            if n == num_nodes-1 then
                node.vm.provision 'ansible' do |ansible|
                    ansible.groups = ansible_groups
                    ansible.playbook = ansible_playbook
                    ansible.extra_vars = ansible_extra_vars
                    ansible.limit = 'all'
                end
            end
        end
    end
end
