# -*- mode: ruby -*-
# vi: set ft=ruby :

# Variables
OPEN5GS_IPv4_ADDR = "192.168.56.100"
UERANSIM_IPv4_ADDR = "192.168.56.120"

def running_rosetta()
  !`sysctl -in sysctl.proc_translated`.strip().to_i.zero?
end

Vagrant.configure("2") do |config|
  config.vagrant.plugins = "vagrant-proxyconf"
  config.vm.box_check_update = false

  # Checking and changing the box based on the architecture 
  # This will be common in all VMs launched
  arch = `arch`.strip()
  if arch == 'arm64' || (arch == 'i386' && running_rosetta())
    config.vm.box = "bento/ubuntu-20.04-arm64"
    config.vm.box_version = "202306.28.0" 
    def_provider = "vmware_desktop"
  else
    config.vm.box = "bento/ubuntu-20.04"
    # TODO : Check which is the latest version works 
    def_provider = "virtualbox"
  end
  
  # The following folders shall be synced and loaded in all VMs 
  config.vm.synced_folder "load-node", "/load-node"
  config.vm.synced_folder "configs", "/configs"
  config.vm.synced_folder "scripts", "/home/vagrant/scripts"

  # VM #1: OPEN5GS 5G core network (All-in-one)
  config.vm.define "open5gs" do |open5gs|
    open5gs.vm.network "private_network", ip: OPEN5GS_IPv4_ADDR

    open5gs.vm.synced_folder "open5gs", "/LoadOpen5gs"
    
    open5gs.vm.provider def_provider do |vb|
     # Customize the amount of cpu & memory on the VM:
     vb.memory = "1024"
     vb.cpus = "2"
    end

    # Open5gs VM bootstrap provisioning with a shell script.
    open5gs.vm.provision "shell", :privileged => true, inline: <<-SHELL
      echo "---------Updating packages--------"
      sudo apt-get update -y 
      
      echo "---------Installing Open5GS--------"
      sudo apt-get install -y wget screen net-tools gnupg cmake python3-pip python3-setuptools python3-wheel ninja-build build-essential flex bison git libsctp-dev libgnutls28-dev libgcrypt-dev libssl-dev libidn11-dev libmongoc-dev libbson-dev libyaml-dev libnghttp2-dev libmicrohttpd-dev libcurl4-gnutls-dev libnghttp2-dev libtins-dev libtalloc-dev meson screen 
      curl -fsSL https://pgp.mongodb.com/server-6.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-6.0.gpg --dearmor
      echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-6.0.gpg] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/6.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list
      sudo apt-get update -y  
      sudo apt-get install -y mongodb-org 
      sudo systemctl enable mongod 
      sudo systemctl start mongod 
      
      sudo cp -r /LoadOpen5gs/ /home/vagrant/open5gs/
      sudo sh /home/vagrant/open5gs/misc/netconf.sh 
      cd /home/vagrant/open5gs
      sudo meson build --prefix=`pwd`/install 
      sudo ninja -C build 
      cd build
      sudo ninja install 
      cd /home/vagrant/
      
      echo "---------Copying to root folders--------"
      sudo cp /home/vagrant/open5gs/install/bin/open* /bin/ 
      sudo cp /home/vagrant/open5gs/misc/db/open5gs-* /bin/
      sudo cp -r /home/vagrant/open5gs/install/var/* /var/ 
      sudo cp -r /home/vagrant/open5gs/install/etc/* /etc/ 
      sudo cp -r /home/vagrant/open5gs/install/lib/* /lib/ 
      echo "---------Finished installing Open5GS ------"
      
      echo "---------Installing Node and and building WebUI------"
      sudo -E bash /load-node/install_node  
      sudo apt-get install -y nodejs yarn  
      
      cd /home/vagrant/open5gs/webui/
      sudo npm ci 
      sudo npm run build  
      
      echo "--------Loading the test configs -------------"
      sudo cp -r /configs/open5gs-config/* /home/vagrant/open5gs/install/etc/
      
      echo "---------Copying all Open5GS services ------"
      sudo cp /configs/open5gs-services/* /usr/lib/systemd/system/

      echo "------- Add a test user in the UDM/UDR database ------"
      sudo open5gs-dbctl add "999700000000001" "465B5CE8B199B49FAA5F0A2EE238A6BC" "E8ED289DEBA952E4283B54E88E6183CA"
      sudo open5gs-dbctl add "999700000000002" "465B5CE8B199B49FAA5F0A2EE238A6BC" "E8ED289DEBA952E4283B54E88E6183CA"    
      sudo open5gs-dbctl add "999700000000003" "465B5CE8B199B49FAA5F0A2EE238A6BC" "E8ED289DEBA952E4283B54E88E6183CA"    
      sudo open5gs-dbctl add "999700000000004" "465B5CE8B199B49FAA5F0A2EE238A6BC" "E8ED289DEBA952E4283B54E88E6183CA"    

      cp /home/vagrant/scripts/* /home/vagrant/
      chmod +x /home/vagrant/*.sh
    SHELL
    
    config.trigger.after :up do |trigger|
      trigger.name = "Finished Message"
      trigger.info = "Machine is up!"
      config.vm.provision :shell, :run => 'always', :privileged => true, inline: <<-SHELL
        # This script will run everytime that the system bootsup 
        echo "--- Adding a route for the UE to have WAN connectivity over SGi/N6 -------"
        sudo sh /home/vagrant/open5gs/misc/netconf.sh
        ### Enable IPv4/IPv6 Forwarding
        sudo sysctl -w net.ipv4.ip_forward=1
        sudo sysctl -w net.ipv6.conf.all.forwarding=1
        ### Add NAT Rule
        sudo iptables -t nat -A POSTROUTING -s 10.45.0.0/16 ! -o ogstun -j MASQUERADE
        sudo ip6tables -t nat -A POSTROUTING -s 2001:db8:cafe::/48 ! -o ogstun -j MASQUERADE
      SHELL
    end
  end

  # VM #2 : UERANSIM Simulator
  config.vm.define "ueransim" do |ueransim|
    ueransim.vm.network "private_network", ip: UERANSIM_IPv4_ADDR

    ueransim.vm.synced_folder "UERANSIM", "/LoadUeransim"
    config.vm.synced_folder "configs", "/configs"

    # Customize the amount of cpu & memory on the VM:
    ueransim.vm.provider def_provider do |vb|
      vb.memory = "1024"
      vb.cpus = "2"
    end

    # #Enable provisioning with a shell script.
    ueransim.vm.provision "shell", :privileged => true, inline:<<-SHELL 
      echo "---------Updating packages --------"
      sudo apt-get update -y 

      echo "---------Installing Dependencies--------"
      sudo apt-get install make g++ libsctp-dev lksctp-tools iproute2 net-tools screen -y 
      sudo snap install cmake --classic 

      echo "---------Compiling UERANSIM--------"
      sudo cp -r /LoadUeransim/ /home/vagrant/UERANSIM/
      cd /home/vagrant/UERANSIM
      sudo make
      cd /home/vagrant/
      echo "---------UERANSIM Compiled--------"

      echo "------- Modify the gNB & UE configuration files  -------"
      sudo cp -r /vagrant/configs/ueransim-config/* /home/vagrant/UERANSIM/config/

      SHELL

  end

end