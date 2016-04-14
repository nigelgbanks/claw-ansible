# -*- mode: ruby -*-
# vi: set ft=ruby :
require 'yaml'

#########################################################
# Looks for a settings file related to a given provider #
#########################################################
def load_settings(file)
  dir = File.dirname(File.expand_path(__FILE__))
  if !File.exist?("#{dir}/vagrant/#{file}.yml")
    FileUtils.cp("#{dir}/vagrant/example.#{file}.yml", "#{dir}/vagrant/#{file}.yml")
    if !File.exist?("#{dir}/vagrant/#{file}.yml")
      raise "Settings file not found! Please copy vagrant/example.#{file}.yml to vagrant/#{file}.yml and try again."
    end
  end
  YAML.load_file "#{dir}/vagrant/#{file}.yml"
end

##################################################
# Maps the ports given in the providers settings #
##################################################
def map_ports(override, settings)
  if (settings.include?('port_mapping'))
    for port_mapping in settings['port_mapping'];
      override.vm.network "forwarded_port", guest: port_mapping['guest'], host: port_mapping['host']
    end
  end
end

#####################################################
# Syncs the folders given in the providers settings #
#####################################################
def sync_folders(override, settings)
  synced_folders = settings.include?('synced_folders') ? settings['synced_folders'] : []

  # Get value of option if defined, or use a default.
  def option_value(synced_folder, option, default)
    return synced_folder.include?(option) ? synced_folder[option] : default;
  end

  # Set the value of an option if defined.
  def set_option(options, option, synced_folder)
    if (synced_folder.include?(option))
      options[option] = synced_folder[option]
    end
  end

  # Set up each sync folder
  for synced_folder in synced_folders;
    options = {
      rsync__auto: "true",
      rsync__exclude: option_value(synced_folder, 'excluded_paths', []),
      create: option_value(synced_folder, 'create', true),
      mount_options: option_value(synced_folder, 'mount_options', [])
    }
    set_option(options, 'type', synced_folder)
    set_option(options, 'owner', synced_folder)
    set_option(options, 'group', synced_folder)
    override.vm.synced_folder synced_folder['local_path'], synced_folder['destination'], options
  end
end

############################
# Install required plugins #
############################
unless Vagrant.has_plugin?("vagrant-triggers") and Vagrant.has_plugin?("vagrant-digitalocean") and Vagrant.has_plugin?("vagrant-aws")

  system("vagrant plugin install vagrant-triggers")
  system("vagrant plugin install vagrant-digitalocean")
  system("vagrant plugin install vagrant-aws")
  puts "Dependencies installed, please try the command again."
  exit
end

VAGRANTFILE_API_VERSION = "2"
Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  ####################
  # Generate Secrets #
  ####################
  config.trigger.before :up do
    run "./scripts/generate-default-configuration"
    run "./scripts/generate-secrets"
  end

  ###################
  # Global Settings #
  ###################
  config.vm.define "islandora-claw"
  config.vm.hostname = "islandora-claw"
  config.ssh.username = "vagrant"
  config.ssh.forward_agent = true

  #####################
  # Global Provisions #
  #####################
  # Fix for the 'no tty' error's displayed during provisioning
  config.vm.provision "fix-no-tty", type: "shell" do |shell|
    shell.privileged = false
    shell.inline = "sudo sed -i '/tty/!s/mesg n/tty -s \\&\\& mesg n/' /root/.profile"
  end

  # Copy .gitconfig if present
  if !File.exist?("~/.gitconfig")
    config.vm.provision "file", source: "~/.gitconfig", destination: "~/.gitconfig"
  end

  # Install Ansible
  config.vm.provision :shell, path: "scripts/install-ansible-on-guest", keep_color: true

  # Install Required Roles
  config.vm.provision :shell, name: "Download required roles", keep_color: true,
                      inline: "ANSIBLE_FORCE_COLOR=true ansible-galaxy install -r /vagrant/requirements.yml"

  # Run Playbook
  config.vm.provision :shell, name: "Download required roles", keep_color: true, privileged: false,
                      inline: "cd /vagrant && ANSIBLE_FORCE_COLOR=true ansible-playbook -i 'localhost, ' --connection=local full-stack.yml"

  ########################
  # Virtual Box Provider #
  ########################
  config.vm.provider :virtualbox do |provider, override|
    settings = load_settings 'virtualbox'
    provider.name = settings['name']
    provider.cpus = settings['cpus']
    provider.memory = settings['memory']
    override.vm.box = settings['box']
    override.vm.synced_folder "./", "/vagrant"
    map_ports override, settings
    sync_folders override, settings
  end

  ##########################
  # Digital Ocean Provider #
  ##########################
  config.vm.provider :digital_ocean do |provider, override|
    settings = load_settings 'digital-ocean'
    provider.name = settings['name']
    provider.image = settings['image']
    provider.ssh_key_name = settings['ssh_key_name']
    provider.token = settings['token']
    provider.region = settings['region']
    provider.size = settings['size']
    override.vm.box = 'digital_ocean'
    override.vm.box_url = "https://github.com/smdahlen/vagrant-digitalocean/raw/master/box/digital_ocean.box"
    override.ssh.private_key_path = settings['private_key_path']
    override.vm.synced_folder "./", "/vagrant",
                              create: true,
                              owner: "vagrant",
                              group: "vagrant"
    sync_folders override, settings
  end

  ################
  # AWS Provider #
  ################
  config.vm.provider :aws do |provider, override|
    # Currently only supports EC2, VPS is not yet exposed.
    settings = load_settings 'aws'
    provider.access_key_id = settings['access_key_id']
    provider.secret_access_key = settings['secret_access_key']
    provider.session_token = settings['session_token']
    provider.keypair_name = settings['keypair_name']
    provider.ami = settings['ami']
    provider.region = settings['region']
    override.vm.box = "dummy"
    override.ssh.username = settings["ssh_username"]
    override.ssh.private_key_path = settings['private_key_path']
    override.vm.synced_folder "./", "/vagrant",
                              create: true,
                              owner: "ubuntu",
                              group: "ubuntu"
    sync_folders override, settings
  end

end
