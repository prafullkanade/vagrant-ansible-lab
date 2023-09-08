require "yaml"
settings = YAML.load_file "settings.yaml"

IP_SECTIONS = settings["network"]["control_ip"].match(/^([0-9.]+\.)([^.]+)$/)
# First 3 octets including the trailing dot:
IP_NW = IP_SECTIONS.captures[0]
# Last octet excluding all dots:
IP_START = Integer(IP_SECTIONS.captures[1])
NUM_WORKER_anodeS = settings["nodes"]["workers"]["count"]

Vagrant.configure("2") do |config|
  config.vm.provision "shell", env: { "IP_NW" => IP_NW, "IP_START" => IP_START, "NUM_WORKER_anodeS" => NUM_WORKER_anodeS }, inline: <<-SHELL
      apt-get update -y
      echo "$IP_NW$((IP_START)) controller-anode" >> /etc/hosts
      for i in `seq 1 ${NUM_WORKER_anodeS}`; do
		#echo "$IP_NW$((IP_START)) controller-anode" >> /etc/hosts
        echo "$IP_NW$((IP_START+i)) worker-anode0${i}" >> /etc/hosts
      done
  SHELL

  if `uname -m`.strip == "aarch64"
    config.vm.box = settings["software"]["box"] + "-arm64"
  else
    config.vm.box = settings["software"]["box"]
  end
  config.vm.box_check_update = true

  config.vm.define "controller" do |controller|
    controller.vm.hostname = "controller-anode"
    controller.vm.network "private_network", ip: settings["network"]["control_ip"]
    if settings["shared_folders"]
      settings["shared_folders"].each do |shared_folder|
        controller.vm.synced_folder shared_folder["host_path"], shared_folder["vm_path"]
      end
    end
    controller.vm.provider "virtualbox" do |vb|
        vb.cpus = settings["nodes"]["control"]["cpu"]
        vb.memory = settings["nodes"]["control"]["memory"]
        if settings["cluster_name"] and settings["cluster_name"] != ""
          vb.customize ["modifyvm", :id, "--groups", ("/" + settings["cluster_name"])]
        end
    end
    controller.vm.provision "shell",
      env: {
        "DNS_SERVERS" => settings["network"]["dns_servers"].join(" "),
        "ENVIRONMENT" => settings["environment"],
        "OS" => settings["software"]["os"]
      },
      path: "scripts/bootstrap.sh"
    #controller.vm.provision "shell",
      #path: "scripts/key_gen.sh"
  end

  (1..NUM_WORKER_anodeS).each do |i|

    config.vm.define "anode0#{i}" do |anode|
      anode.vm.hostname = "worker-anode0#{i}"
      anode.vm.network "private_network", ip: IP_NW + "#{IP_START + i}"
      if settings["shared_folders"]
        settings["shared_folders"].each do |shared_folder|
          anode.vm.synced_folder shared_folder["host_path"], shared_folder["vm_path"]
        end
      end
      anode.vm.provider "virtualbox" do |vb|
          vb.cpus = settings["nodes"]["workers"]["cpu"]
          vb.memory = settings["nodes"]["workers"]["memory"]
          if settings["cluster_name"] and settings["cluster_name"] != ""
            vb.customize ["modifyvm", :id, "--groups", ("/" + settings["cluster_name"])]
          end
      end
      anode.vm.provision "shell",
        env: {
          "DNS_SERVERS" => settings["network"]["dns_servers"].join(" "),
          "ENVIRONMENT" => settings["environment"],
          "OS" => settings["software"]["os"]
        },
        path: "scripts/bootstrap.sh"
    end
  end
end 
