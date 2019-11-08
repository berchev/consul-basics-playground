Vagrant.configure("2") do |config|
    (1..3).each do |i|
      config.vm.define vm_name="server#{i}" do |node|
        node.vm.box = "berchev/xenial64"
        node.vm.hostname = vm_name
        node.vm.provision :shell, path: "scripts/provision.sh"
        node.vm.network :forwarded_port, guest: 8500, host: 8500 + i 
        node.vm.network "private_network", ip: "192.168.10.#{i}1"
      end
    end 
end