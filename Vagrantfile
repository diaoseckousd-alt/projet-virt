Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"

  # ==========================================
  # VM 1 : firewall
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

    # Provisioning Shell : Réseau & Sécurité Prof
    fw.vm.provision "shell", inline: <<-SHELL
      rm -f /etc/resolv.conf && echo 'nameserver 8.8.8.8' > /etc/resolv.conf
      sysctl -w net.ipv4.ip_forward=1
      iptables -F
      iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
      iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
      iptables -A FORWARD -i enp0s9 -o enp0s8 -j ACCEPT
      iptables -A FORWARD -i enp0s3 -o enp0s8 -p tcp --dport 80 -j ACCEPT
      iptables -A FORWARD -i enp0s8 -o enp0s9 -j DROP
      iptables -P FORWARD DROP
    SHELL

    # Provisioning Ansible Interne
    fw.vm.provision "ansible_local" do |ansible|
      ansible.playbook = "ansible/firewall.yml"
    end

    # AUTOMATISATION WINDOWS (Tunnel + Runner)
    fw.vm.trigger.after :up do |trigger|
      trigger.info = "Ouverture des outils Windows..."
      trigger.run = { inline: "start powershell -NoExit -Command \"ssh -i .vagrant/machines/firewall/virtualbox/private_key -L 9090:192.168.100.10:80 vagrant@127.0.0.1 -p 2222\"" }
      trigger.run = { inline: "start powershell -NoExit -Command \"cd actions-runner; .\\run.cmd\"" }
    end
  end

  # ==========================================
  # VM 2 : webserver
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
    SHELL

    web.vm.provision "ansible_local" do |ansible|
      ansible.playbook = "ansible/webserver.yml"
    end
  end

  # ==========================================
  # VM 3 : dbserver
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
    SHELL

    db.vm.provision "ansible_local" do |ansible|
      ansible.playbook = "ansible/dbserver.yml"
    end
  end
end