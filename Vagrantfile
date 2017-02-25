# -*- mode: ruby -*-
# vi: set ft=ruby :

domain          = "test.dev"
setup_complete  = false
deploy_dir      = "/var/www/html"

# NOTE: currently using the same OS for all boxen
OS="centos" # "debian" || "centos"

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure(2) do |config|
  if OS=="debian"
    config.vm.box = "debian/jessie64"
  elsif OS=="centos"
    config.vm.box = "centos/7"
  else
    puts "you must set the OS variable to a valid value before continuing"
    exit
  end

  {
    # 'solr'  => '10.7.7.103',
    # 'db'    => '10.7.7.102',
    'sufia'   => '10.7.7.101'
  }.each do |short_name, ip|
    config.vm.define short_name do |host|
      host.vm.network 'private_network', ip: ip
      host.vm.hostname = "#{short_name}.#{domain}"
      # presumes installation of https://github.com/cogitatio/vagrant-hostsupdater on host
      host.hostsupdater.aliases = ["#{short_name}"]
      # avoiding "Authentication failure" issue
      host.ssh.insert_key = false
      host.vm.synced_folder ".", "/vagrant", disabled: true

      host.vm.provider "virtualbox" do |vb|
        vb.name = "#{short_name}.#{domain}"
        vb.memory = 2048
        vb.linked_clone = true

        # NOTE: will need refactoring if separating into multiple VMs
        # port forwarding solr:
        host.vm.network "forwarded_port", guest: 8983, host: 8983, auto_correct: true
        # port forwarding fedora (via tomcat):
        host.vm.network "forwarded_port", guest: 8080, host: 8888, auto_correct: true
        # port forwarding sufia (puma):
        host.vm.network "forwarded_port", guest: 3000, host: 3000, auto_correct: true
        # port forwarding sufia (http):
        host.vm.network "forwarded_port", guest: 80, host: 8080, auto_correct: true
        # port forwarding sufia (https):
        host.vm.network "forwarded_port", guest: 443, host: 4443, auto_correct: true
      end

      if short_name == "sufia" # last in the list
        setup_complete = true
      end

      if setup_complete
        if OS=="centos"
          # workaround for https://github.com/mitchellh/vagrant/issues/8142
          host.vm.provision "shell",
            inline: "sudo service network restart"
        end

        host.vm.provision "ansible" do |ansible|
          ansible.galaxy_role_file = "requirements.yml"
          ansible.inventory_path = "inventory/vagrant"
          ansible.playbook = "setup.yml"
          ansible.limit = "all"
          # ansible.verbose ="vvv"
        end

        # NOTE: nfs synced_folder is broken in vagrant 1.9.1.
        # must downgrade to 1.9.0 for this to work
        # https://github.com/mitchellh/vagrant/issues/8138
        host.vm.synced_folder "project-code", "#{deploy_dir}", type: 'nfs', mount_options: ['rw', 'vers=3', 'tcp', 'fsc' ,'actimeo=1'], map_uid: 0, map_gid: 0
      end
    end
  end
end
