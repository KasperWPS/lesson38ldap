# -*- mode: ruby -*- 
# vi: set ft=ruby : vsa
Vagrant.require_version ">= 2.2.17"

class Hash
  def rekey
  t = self.dup
  self.clear
  t.each_pair{|k, v| self[k.to_sym] = v}
    self
  end
end

require 'json'

f = JSON.parse(File.read(File.join(File.dirname(__FILE__), 'config.json')))
# Локальная переменная PATH_SRC для монтирования
$PathSrc = ENV['PATH_SRC'] || "."

Vagrant.configure(2) do |config|
  if Vagrant.has_plugin?("vagrant-vbguest")
    config.vbguest.auto_update = false
  end

  # включить переадресацию агента ssh
  config.ssh.forward_agent = true
  # использовать стандартный для vagrant ключ ssh
  config.ssh.insert_key = false

  last_vm = f[(f.length)-1]['name']

  f.each do |g|

    config.vm.define g['name'] do |s|
      s.vm.box = g['box']
      s.vm.hostname = g['name']

      if g['private_network']
        g['private_network'].each do |ni|
          s.vm.network "private_network", **ni.rekey
        end
      end

#      if g['forward_port']
#        s.vm.network 'forwarded_port', guest: g['forward_port'], host: g['forward_port'], auto_correct: true
#      end

      s.vm.synced_folder $PathSrc, "/vagrant", disabled: g['no_share']

      s.vm.provider :virtualbox do |virtualbox|
        virtualbox.customize [
          "modifyvm",             :id,
          "--audio",              "none",
          "--cpus",               g['cpus'],
          "--memory",             g['memory'],
          "--graphicscontroller", "VMSVGA",
          "--vram",               "64"
        ]

        attachController = false

        if g['disks']
          g['disks'].each do |dname, dconf|
            unless File.exist? (dconf['dfile'])
              attachController = true
              virtualbox.customize [
                'createhd',
                '--filename', dconf['dfile'],
                '--variant',  'Fixed',
                '--size',     dconf['size']
              ]
            end
          end
          if attachController == true
            virtualbox.customize [
              "storagectl", :id,
              "--name",     "SAS Controller",
              "--add",      "sas"
            ]
          end
          g['disks'].each do |dname, dconf|
            virtualbox.customize [
              'storageattach', :id,
              '--storagectl',  'SAS Controller',
              '--port',        dconf['port'],
              '--device',      0,
              '--type',        'hdd',
              '--medium',      dconf['dfile']
            ]
          end
        end
        virtualbox.gui = g['gui']
        virtualbox.name = g['name']
#        if g['nic_type']
#          virtualbox.default_nic_type = g['nic_type']
#        end
      end
      s.vm.provision "ansible" do |ansible|
        ansible.playbook = "provisioning/playbook.yml"
        ansible.inventory_path = "provisioning/hosts"
        ansible.host_key_checking = "false"
        ansible.become = "true"
      end
    end
  end
end
