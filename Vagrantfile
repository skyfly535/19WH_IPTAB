# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
    :inetRouter => { # задаем параметры и конфигурацию сетевых адаптеров для inetRouter
        :box_name => "centos/7",
        :net => [
            {ip: '192.168.255.1', adapter: 2, netmask: "255.255.255.240", virtualbox__intnet: "inet-router-net"},
            {ip: '192.168.56.10', adapter: 8},
        ]
    },
    :inetRouter2 => {  # задаем параметры и конфигурацию сетевых адаптеров для inetRouter2
        :box_name => "centos/7",
        :net => [
            {ip: '192.168.255.3', adapter: 2, netmask: "255.255.255.240", virtualbox__intnet: "inet-router-net"},
            {ip: '192.168.56.11', adapter: 8},
         ]
    },
    :centralRouter => { # задаем параметры и конфигурацию сетевых адаптеров для centralRouter
        :box_name => "centos/7",
        :net => [
            {ip: '192.168.255.2', adapter: 2, netmask: "255.255.255.240", virtualbox__intnet: "inet-router-net"},
            {ip: '192.168.0.1',   adapter: 3, netmask: "255.255.255.252", virtualbox__intnet: "central-router-net"},
            {ip: '192.168.56.12', adapter: 8},
        ]
    },
    :centralServer => { # задаем параметры и конфигурацию сетевых адаптеров для centralServer
        :box_name => "centos/7",
        :net => [
            {ip: '192.168.0.2',   adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "central-router-net"},
            {ip: '192.168.56.13', adapter: 8},
        ]
    },
}

Vagrant.configure("2") do |config|

    MACHINES.each do |boxname, boxconfig|
        config.gatling.rsync_on_startup = false
        config.vm.define boxname do |box|

            box.vm.provision "shell", run: "always", inline: <<-SHELL
                sudo -i # перходим в режим sudo
                systemctl stop NetworkManager    # останавливаем службу NetworkManager
                systemctl disable NetworkManager # убираем из автозагрузки
                systemctl enable network.service # ставим в автозагрузку network.service
                systemctl start network.service
                systemctl stop firewalld # останавливаем firewalld, чтоб запустить iptables
                systemctl mask firewalld # скрываем сервис, чтобы другие скрипты не смогли его запустить
                yum install -y iptables-services # устанавливаем iptables
                systemctl start iptables # запускаем iptables
                systemctl enable iptables # добавляем iptables в автозагрузку
                yum install -y traceroute # ставим traceroute для проверки
                yum install -y nano
            SHELL

            case boxname.to_s
            when "inetRouter" # конфигурируем inetRouter
              box.vm.provision "file", source: "/home/roman/project/iptables.rules", destination: "iptables.rules" # переносим с хостовой машины файл конфигурации iptables с реализованным реализовать knocking port
              box.vm.provision "shell", run: "always", inline: <<-SHELL
                sed -i 's/^PasswordAuthentication.*$/PasswordAuthentication yes/' /etc/ssh/sshd_config #Разрешаем подключение пользователей по SSH с использованием пароля
                systemctl restart sshd.service #Перезапуск службы SSHD
                echo "net.ipv4.ip_forward=1" >>/etc/sysctl.conf # Настройка маршрутизации транзитных пакетов (не изменяется после перезапуска) 
                touch /etc/sysconfig/network-scripts/route-eth1 # создаем файл (интерфейса eth1 - вся сеть) для хранения обраных маршрутов (не изменяется после перезапуска)
              SHELL
            end

            case boxname.to_s
            when "centralRouter" # конфигурируем inetRouter2
              box.vm.provision "file", source: "/home/roman/project/iptables", destination: "iptables" # переносим с хостовой машины файл конфигурации iptables
              box.vm.provision "file", source: "/home/roman/project/knock.sh", destination: "knock.sh" # переносим с хостовой машины файл скрипта для подключения к inetRouter через knock
              box.vm.provision "shell", run: "always", inline: <<-SHELL
                echo "net.ipv4.ip_forward=1" >>/etc/sysctl.conf # Настройка маршрутизации транзитных пакетов (не изменяется после перезапуска)
                chmod +x /home/vagrant/knock.sh # даем файлу knock скрипта права на исполнение
              SHELL
            end

            case boxname.to_s
            when "inetRouter2"
              box.vm.provision "file", source: "/home/roman/project/iptables", destination: "iptables" # переносим с хостовой машины файл конфигурации iptables
              box.vm.provision "shell", run: "always", inline: <<-SHELL
                echo "net.ipv4.ip_forward=1" >>/etc/sysctl.conf # Настройка маршрутизации транзитных пакетов (не изменяется после перезапуска)
                touch /etc/sysconfig/network-scripts/route-eth1 # создаем файл (интерфейса eth1 - вся сеть) для хранения обраных маршрутов (не изменяется после перезапуска)
              SHELL
              box.vm.network 'forwarded_port', guest: 8080, host: 8080, host_ip: '127.0.0.1' # форвардим порты на хост машину для проверки проброса nginx сервера с centralServer через inetRouter2
            end

            case boxname.to_s
            when "centralServer"
              box.vm.provision "file", source: "/home/roman/project/iptables", destination: "iptables" # переносим с хостовой машины файл конфигурации iptables
            end

            config.vm.provider "virtualbox" do |v|
                v.memory = 256
                v.cpus = 1
            end

            box.vm.box = boxconfig[:box_name]
            box.vm.host_name = boxname.to_s

            boxconfig[:net].each do |ipconf|
                box.vm.network "private_network", ipconf
            end

            if boxconfig.key?(:public)
                box.vm.network "public_network", boxconfig[:public]
            end

            box.vm.provision "shell", inline: <<-SHELL
                mkdir -p ~root/.ssh
                cp ~vagrant/.ssh/auth* ~root/.ssh
            SHELL

            box.vm.provision "ansible" do |ansible|  # вызываем playbook для развертывания нужной инфраструктуры на машинах
              ansible.verbose = "v"
              ansible.playbook = "routing.yml" # файл playbook
            end

        end
    end
end