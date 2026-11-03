# coding: utf-8
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  config.vm.box = "debian/bookworm64"
  
  config.ssh.insert_key = false
  
  config.vm.provider "virtualbox" do |vb|
    vb.gui = false
    vb.linked_clone = true
    vb.memory = "256"
  end

  # Común a todas las máquinas
  
  config.vm.provision "shell", inline: <<-SHELL
    apt-get update -y
  SHELL

  # ns1.olimpo.test
  
  config.vm.define "dns" do |dns|

    dns.vm.hostname = "ns1.olimpo.test"
    
    dns.vm.network :private_network, ip: "192.168.57.10",
                   hostname: true

    # Instalar servidor DNS
    
    dns.vm.provision "bind9", type: "shell",
                     inline: "apt-get install -y bind9 bind9-dnsutils"

    dns.vm.provision "dns-resolv", type: "shell",
                     inline: "cp -v /vagrant/files/ns1.olimpo.test/resolv.conf /etc"

    # Configurar servidor DNS
    
    dns.vm.provision "shell", name: "dns-provision",
                     path: "dns-provision.sh"
    
  end # dns

  # mercurio.olimpo.test

  config.vm.define "web" do |mercurio|
    
    mercurio.vm.hostname = "mercurio.olimpo.test"
    
    mercurio.vm.network :private_network, ip: "192.168.57.11",
                        hostname: true

    mercurio.vm.provision "shell", name: "mercurio-resolv",
                          inline: "cp -v /vagrant/files/mercurio.olimpo.test/resolv.conf /etc"
    
    # Instalar servidor web
    
    mercurio.vm.provision "shell", name: "apache2",
                          inline: "apt-get install -y apache2"

    # Ficheros del servidor apolo
    
    mercurio.vm.synced_folder "files/apolo.olimpo.test", "/var/www/apolo.olimpo.test",
                              group: "www-data", owner: "www-data"

    # Ficheros del servidor atenea
    
    mercurio.vm.synced_folder "files/atenea.olimpo.test", "/var/www/atenea.olimpo.test",
                              group: "www-data", owner: "www-data"

    # Configurar servidor web
    
    mercurio.vm.provision "shell", name: "apache2-provision",
                          path: "apache2-provision.sh"

  end # mercurio
   
end 
