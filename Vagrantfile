Vagrant.configure("2") do |config|
  # Base VM OS configuration.
  config.vm.box = "ubuntu-22.04"
  config.vm.provider :virtualbox do |v|
    v.memory = 1512
    v.cpus = 2
  end

  # Define two VMs with static private IP addresses.
  boxes = [
    { :name => "web",
      :ip => "192.168.56.10",
    },
    { :name => "log",
      :ip => "192.168.56.15",
    }
  ]
  # Provision each of the VMs.
  boxes.each do |opts|
    config.vm.define opts[:name] do |config|
      config.vm.hostname = opts[:name]
      config.vm.network "private_network", ip: opts[:ip]
    config.vm.provision "shell", inline: <<-SHELL
         mkdir -p ~root/.ssh
         cp ~vagrant/.ssh/auth* ~root/.ssh
		 sudo -s
		 timedatectl set-timezone Europe/Moscow
		 date
          SHELL
    end
    config.vm.provision 'ansible' do |ansible|
      ansible.playbook = 'playbook.yml'
    end
  end
end
