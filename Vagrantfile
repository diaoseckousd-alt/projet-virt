Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"

  # ==========================================
  # VM 1 : firewall (Routeur & Gateway)
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

    fw.vm.provision "shell", inline: <<-SHELL
      sysctl -w net.ipv4.ip_forward=1
      apt-get update -y && apt-get install -y ansible
      iptables -F
      iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
      iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
      iptables -A FORWARD -i enp0s9 -o enp0s8 -j ACCEPT
      iptables -A FORWARD -i enp0s3 -o enp0s8 -p tcp --dport 80 -j ACCEPT
    SHELL

    fw.vm.provision "ansible_local" do |ansible|
      ansible.playbook = "ansible/firewall.yml"
      ansible.install = false
    end

    # Trigger unique pour lancer les outils Windows
    fw.trigger.after :up, :provision do |t|
      t.info = "Lancement du Tunnel SSH et du Runner GitHub..."
      project_path = File.expand_path(File.dirname(__FILE__))
      
      # Tunnel vers Nginx (Port 80) qui renverra vers Node.js
      t.run = { inline: "cmd /c start powershell -NoExit -Command \"ssh -o StrictHostKeyChecking=no -i .vagrant/machines/firewall/virtualbox/private_key -L 9090:192.168.100.10:80 vagrant@127.0.0.1 -p 2222\"" }
      
      # GitHub Runner
      t.run = { inline: "cmd /c start powershell -NoExit -Command \"cd '#{project_path}/actions-runner'; .\\run.cmd\"" }
    end
  end

  # ==========================================
  # VM 2 : webserver (DMZ - Nginx + Node.js)
  # ==========================================
  config.vm.define "webserver" do |web|
    web.vm.hostname = "web-server"
    web.vm.network "private_network", ip: "192.168.100.10", virtualbox__intnet: "dmz-net"
    
    web.vm.provision "shell", inline: <<-SHELL
      ip route del default || true
      ip route add default via 192.168.100.1
      echo 'nameserver 8.8.8.8' > /etc/resolv.conf
    SHELL

    web.vm.provision "ansible_local" do |ansible|
      ansible.playbook = "ansible/webserver.yml"
    end
  end

  # ==========================================
  # VM 3 : dbserver (LAN - MariaDB)
  # ==========================================
  config.vm.define "dbserver" do |db|
    db.vm.hostname = "db-server"
    db.vm.network "private_network", ip: "192.168.10.10", virtualbox__intnet: "lan-net"
    
    db.vm.provision "shell", inline: <<-SHELL
      ip route del default || true
      ip route add default via 192.168.10.1
    SHELL

    db.vm.provision "ansible_local" do |ansible|
      ansible.playbook = "ansible/dbserver.yml"
    end
  end
end