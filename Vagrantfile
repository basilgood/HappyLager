# -*- mode: ruby -*-
# vi: set ft=ruby :

def required(*plugins)
  if Vagrant.in_installer?
    for plugin in plugins
      unless Vagrant.has_plugin?(plugin)
        raise Vagrant::Errors::PluginNotInstalled.new :name => plugin
      end
    end
  end
end

#Project name used for database name,user & pass
ip        = "192.168.30.150"
project  	= "happylager"
windows 	=	Vagrant::Util::Platform.windows?

#folder
nfs 		=	[ 'rw', 'vers=3', 'tcp', 'fsc','actimeo=1']
smb 		= [ 'file_mode=0777,dir_mode=0777']
share 	= [ './', '/vagrant']
nfsf 		= share + [type: 'nfs', :mount_options =>nfs]
smbf 		= share + [type: 'smb', :mount_options =>smb ]

#Ansible
ansibl = windows ? :ansible_local : :ansible

Vagrant.configure(2) do |config|
  config.vm.box = "ubuntu/trusty64"
  required "vagrant-hostsupdater"

  #LXC - Linux only
  config.vm.provider :lxc do |lxc,ovrd|
    ovrd.vm.box = 'fgrehm/trusty64-lxc'
    lxc.customize 'cgroup.memory.limit_in_bytes', '512M'
    lxc.customize 'aa_allow_incomplete', 1
    #lxc.customize 'network.ipv4',"#{ip}/24";
    #lxc.customize 'network.ipv4.gateway','auto';
    lxc.container_name = project
    ovrd.vm.synced_folder *share
  end

  #virtualbox
  config.vm.provider :virtualbox do |vb,ovrd|
    vb.customize ["setextradata", :id, "VBoxInternal2/SharedFoldersEnableSymlinksCreate/vagrant", "1"]
    vb.customize ["modifyvm", :id, "--pae", "on"]
    vb.customize ["modifyvm", :id, "--vram", "128"]
    vb.customize ["modifyvm", :id, "--cpus", "2"]
    #vb.customize ["setextradata", :id, "GUI/ScaleFactor", "1.0"]

    vb.memory = 1024
  end

  #hyperv
  config.vm.provider "hyperv" do |hv,ovrd|
    hv.vmname = project
    ovrd.vm.synced_folder(*smbf)
  end

  #Multimachine: the develop machine
  config.vm.define :develop, primary: true do |dev|
    dev.vm.provider :virtualbox
    dev.vm.provider :virtualbox do |vb, ovrd|
      ovrd.vm.synced_folder *(windows ? share : nfsf)
    end
    dev.vm.hostname = "#{project}.local"
    dev.vm.network :private_network, ip: ip, lxc__bridge_name: 'vlxcbr1'
    dev.vm.network "forwarded_port", guest: 80, host: 8080
    dev.vm.provision 'ansible' do |ansible|
      ansible.playbook = 'provision/ansible/develop.yml'
      ansible.galaxy_role_file = 'provision/ansible/galaxy.yml'
      ansible.extra_vars = { project: project}
    end
  end

  #Multimachine: the OSX machine
  config.vm.define "osx", autostart: false do |osx|
    osx.vm.box = "http://files.dryga.com/boxes/osx-sierra-0.3.1.box"
    osx.vm.network :private_network, ip: '10.10.10.10'
    osx.vm.provider "virtualbox" do |vb, ovrd|
      vb.customize ["modifyvm", :id, "--vram", "128"]
      vb.customize ["modifyvm", :id, "--cpus", "2"]
      vb.customize ["setextradata", :id, "VBoxInternal2/EfiGopMode", 3]
      ovrd.vm.synced_folder *(share + [disabled: true])
      vb.gui = true
    end
    osx.ssh.forward_agent = true
  end

  #Multimachine: the Windows machine
  config.vm.define 'windows', autostart: false do |win|
    win.vm.communicator = "winrm"
    win.vm.box = "plumelo/whatslike"
    win.vm.network :private_network, ip: '10.10.10.10'
    win.vm.provider "virtualbox" do |vb, ovrd|
      vb.gui = true
      ovrd.vm.synced_folder *share
    end
    win.vm.provision ansibl do |ansible|
			ansible.playbook = 'provision/ansible/windows.yml'
			ansible.extra_vars = { project: project, ip:ip}
		end
    # win.vm.provision "shell", path: "path/to/script"
  end

end
