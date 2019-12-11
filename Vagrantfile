# -*- mode: ruby -*-
# vi: set ft=ruby :
require 'yaml'
VAGRANTFILE_API_VERSION = '2'

config_file=File.expand_path(File.join(File.dirname(__FILE__), 'vagrant_variables.yml'))

settings=YAML.load_file(config_file)

BOX                       = "generic/ubuntu1804"
MASTERS_VMS               = (ENV['MASTERS_VMS'] || settings['masters_vms']).to_i
WORKERS_VMS               = (ENV['WORKERS_VMS'] || settings['workers_vms']).to_i
CPUS                      = (ENV['CPUS'] || settings['cpus']).to_i
MEMORY                    = (ENV['MEMORY'] || settings['memory']).to_i
NETWORK_PREFIX            = ENV['NETWORK_PREFIX'] || "192.168.100"
DEBUG                     = ENV['DEBUG'] || settings['debug']
KEYMAP                    = ENV['KEYMAP'] || settings['keymap']
K3S_CLUSTER_SECRET        = ENV['K3S_CLUSTER_SECRET'] || settings['k3s_cluster_secret']
NET_INTERFACE             = ENV['NET_INTERFACE'] || settings['net_interface']
CONSUL_ENCRYPT_SECUREPASS = ENV['CONSUL_ENCRYPT_SECUREPASS'] || settings['consul_encrypt_securepass']

# --- Check for missing plugins
required_plugins = %w( vagrant-alpine vagrant-timezone vagrant-mutate vagrant-libvirt )
plugin_installed = false
required_plugins.each do |plugin|
  unless Vagrant.has_plugin?(plugin)
    system "vagrant plugin install #{plugin}"
    plugin_installed = true
  end
end
# --- If new plugins installed, restart Vagrant process
if plugin_installed === true
  exec "vagrant #{ARGV.join' '}"
end

provision = <<SCRIPT
  sed -i 's/ChallengeResponseAuthentication no/ChallengeResponseAuthentication yes/g' /etc/ssh/sshd_config
  systemctl restart ssh
SCRIPT

ansible_provision = proc do |ansible|
  ansible.playbook = 'k3s-consul.yml'
  ansible.sudo = true
  ansible.groups = {
    'k3sm'  => (0..MASTERS_VMS).map { |j| "k3sm#{j}" },
  }
  ansible.extra_vars = {
    k3s_cluster_secret: K3S_CLUSTER_SECRET,
    consul_encrypt_securepass: CONSUL_ENCRYPT_SECUREPASS,
    net_interface: NET_INTERFACE,
  }
  if DEBUG then
    ansible.verbose = '-vvvv'
  end
  ansible.limit = 'all'
end

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.provider :libvirt do |v|
    v.memory = MEMORY
    v.video_vram = 32
    v.cpus = CPUS
    v.cpu_mode = "host-model"
    v.nested = true
    v.keymap = KEYMAP
  end

  config.vm.box = BOX
  config.vm.provision "shell", inline: provision
  config.timezone.value = :host

  # master server
  (1..MASTERS_VMS).each do |i|
    config.vm.define "k3sm#{i}" do |kmaster|
      kmaster.vm.hostname = "k3sm#{i}"
      kmaster.vm.network :private_network, ip: "#{NETWORK_PREFIX}.#{200+i}"
      # Run the provisioner after the last machine comes up
      kmaster.vm.provision 'ansible', &ansible_provision if i == MASTERS_VMS
    end
  end

  # slave server
  (1..WORKERS_VMS).each do |i|
    config.vm.define "k3sw#{i}" do |knode|
      knode.vm.hostname = "k3sw#{i}"
      knode.vm.network :private_network, ip: "#{NETWORK_PREFIX}.#{100+i}"
    end
  end
end