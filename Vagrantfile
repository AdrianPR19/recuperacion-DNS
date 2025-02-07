Vagrant.configure("2") do |config|
  config.vm.box = "debian/bullseye64"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "256"
    vb.linked_clone = true
  end

 
  config.vm.provision "update", type: "shell", inline: <<-SHELL
    apt-get update 
  SHELL

 
  # Servidor atlas
  
  config.vm.define "atlas" do |atlas|
    atlas.vm.hostname = "atlas"  
    atlas.vm.network "private_network", ip: "192.168.56.10"  

    atlas.vm.provision "bind9-install", type: "shell", inline: <<-SHELL
        apt-get install -y bind9 bind9-utils bind9-doc
    SHELL
  end

  
  # Servidor ceo
  
  config.vm.define "ceo" do |ceo|
    ceo.vm.hostname = "ceo"  
    ceo.vm.network "private_network", ip: "192.168.56.11"  

    ceo.vm.provision "bind9-install", type: "shell", inline: <<-SHELL
        apt-get install -y bind9 bind9-utils bind9-doc
    SHELL
  end
end