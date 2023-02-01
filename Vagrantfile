# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :'server.otus.test' => {
        :box_name => "centos/7",
        :public => [ {adapter: 2, name: "vboxnet0", ip: '192.168.56.10', netmask: "255.255.255.0"} ],
        :private => [
          {adapter: 3, ip: '192.168.57.10', netmask: "255.255.255.0", virtualbox__intnet: "clients1-net"},
          # {adapter: 4, ip: '192.168.58.10', netmask: "255.255.255.0", virtualbox__intnet: "clients2-net"}
        ]
  },
  :'client1.otus.test' => {
        :box_name => "centos/7",
        :private => [
          {adapter: 2, ip: '192.168.57.11', netmask: "255.255.255.0", virtualbox__intnet: "clients1-net"}
        ]
  },
  # :'client2.org.test' => {
  #       :box_name => "centos/7",
  #       :private => [
  #         {adapter: 2, ip: '192.168.58.11', netmask: "255.255.255.0", virtualbox__intnet: "clients2-net"}
  #       ]
  # }
}

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
        config.vm.define boxname do |box|

          box.vm.box = boxconfig[:box_name]
          box.vm.host_name = boxname.to_s

          if boxconfig.key?(:public)
            boxconfig[:public].each do |ipconf|
              #box.vm.network "public_network", boxconfig[:public]
              box.vm.network "private_network", ipconf
            end
          end

          boxconfig[:private].each do |ipconf|
            box.vm.network "private_network", ipconf
          end

          case boxname.to_s
          when "server.otus.test"
            box.vm.provider :virtualbox do |vb|
              vb.customize [
                'modifyvm', :id,
                '--memory', '2048',
                '--cpus', '3',
              ]
            end
          when "client1.otus.test"
            box.vm.provider :virtualbox do |vb|
              vb.customize [
                'modifyvm', :id,
                '--memory', '2048',
                '--cpus', '1',
              ]
            end
          when "client2.org.test"
            box.vm.provider :virtualbox do |vb|
              vb.customize [
                'modifyvm', :id,
                '--memory', '2048',
                '--cpus', '1',
              ]
            end
          end

          # box.vm.network "private_network", ip: boxconfig[:ip_addr]
          # box.vm.provider :virtualbox do |vb|
          #   vb.customize ["modifyvm", :id, "--memory", "2048"]
          # end
          box.vm.provision "ansible" do |ansible|
          #   ansible.become = true
            # ansible.inventory_path = "ansible-freeipa/inventory/hosts"
            ansible.verbose = "vvv"
            ansible.playbook = "ansible/provision.yml"
          end
      end
  end
end