Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"

  # ==========================================
  # VM 1 : firewall (Passerelle / Pare-feu)
  # ==========================================
  config.vm.define "firewall" do |fw|
    fw.vm.hostname = "firewall"
    fw.vm.network "private_network", ip: "192.168.100.1", virtualbox__intnet: "dmz-net"
    fw.vm.network "private_network", ip: "192.168.10.1", virtualbox__intnet: "lan-net"

    fw.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      vb.name = "firewall"
      vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
    end

    # Configuration Réseau + Routage + IPTables
    fw.vm.provision "shell", inline: <<-SHELL
      rm -f /etc/resolv.conf && echo 'nameserver 8.8.8.8' > /etc/resolv.conf
      sysctl -w net.ipv4.ip_forward=1
      apt-get update -y && apt-get install -y ansible
      
      # Règles de filtrage
      iptables -F
      iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
      iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
      iptables -A FORWARD -i enp0s9 -o enp0s8 -j ACCEPT                # LAN -> WEB
      iptables -A FORWARD -i enp0s3 -o enp0s8 -p tcp --dport 80 -j ACCEPT # Internet -> WEB
      iptables -A FORWARD -i enp0s8 -o enp0s9 -j DROP                  # WEB -> LAN
      iptables -P FORWARD DROP
    SHELL

    fw.vm.provision "ansible_local" do |ansible|
      ansible.playbook = "ansible/firewall.yml"
      ansible.install = false # Déjà installé par le shell
    end
  end

  # ==========================================
  # VM 2 : webserver (Serveur Web DMZ)
  # ==========================================
  config.vm.define "webserver" do |web|
    web.vm.hostname = "web-server"
    web.vm.network "private_network", ip: "192.168.100.10", virtualbox__intnet: "dmz-net"

    web.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      vb.name = "webserver"
    end

    web.vm.provision "shell", inline: <<-SHELL
      rm -f /etc/resolv.conf && echo 'nameserver 8.8.8.8' > /etc/resolv.conf
      ip route del default
      ip route add default via 192.168.100.1
      apt-get update -y && apt-get install -y ansible
    SHELL

    web.vm.provision "ansible_local" do |ansible|
      ansible.playbook = "ansible/webserver.yml"
      ansible.install = false
    end
  end

  # ==========================================
  # VM 3 : dbserver (Serveur BD LAN)
  # ==========================================
  config.vm.define "dbserver" do |db|
    db.vm.hostname = "db-server"
    db.vm.network "private_network", ip: "192.168.10.10", virtualbox__intnet: "lan-net"

    db.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      vb.name = "dbserver"
    end

    db.vm.provision "shell", inline: <<-SHELL
      rm -f /etc/resolv.conf && echo 'nameserver 8.8.8.8' > /etc/resolv.conf
      ip route del default
      ip route add default via 192.168.10.1
      apt-get update -y && apt-get install -y ansible
    SHELL

    db.vm.provision "ansible_local" do |ansible|
      ansible.playbook = "ansible/dbserver.yml"
      ansible.install = false
    end
  end

  # ==========================================
  # TRIGGERS : Tunnel + Runner (Windows)
  # ==========================================
  config.trigger.after :up do |t|
    t.info = "Lancement automatique des outils Windows..."
    
    # Correction Tunnel SSH (Ouvre une fenêtre PowerShell)
    t.run = { inline: "cmd /c start powershell -NoExit -Command \"ssh -i .vagrant/machines/firewall/virtualbox/private_key -L 9090:192.168.100.10:80 vagrant@127.0.0.1 -p 2222\"" }
    
    # Correction Runner GitHub (Ouvre une fenêtre PowerShell)
    t.run = { inline: "cmd /c start powershell -NoExit -Command \"cd actions-runner; .\\run.cmd\"" }
  end

end