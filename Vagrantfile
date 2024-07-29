Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64" 
  config.vm.define "server" do |server| 
    server.vm.hostname = "server.loc" 
    server.vm.network "private_network", ip: "192.168.56.10" 
  end 
  
  config.vm.define "client" do |client| 
    client.vm.hostname = "client.loc" 
    client.vm.network "private_network", ip: "192.168.56.20" 
  end
  
  ssh_pub_key = File.readlines("../id_rsa.pub").first.strip
  config.vm.provision "shell", inline: <<-SHELL
    echo #{ssh_pub_key} >> ~vagrant/.ssh/authorized_keys
    echo #{ssh_pub_key} >> ~root/.ssh/authorized_keys
    sudo sed -i 's/\#PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
    systemctl restart sshd
  SHELL
end
  