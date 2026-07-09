Vagrant.configure("2") do |config|
  # --- Hokage-DC01: Konohagakure's domain controller (noad.local) ---
  config.vm.define "hokage-dc01" do |m|
    m.vm.box = "gusztavvargadr/windows-server-2022-standard"
    m.vm.hostname = "hokage-dc01"
    m.vm.network "private_network", ip: "192.168.56.10"
    m.vm.provider "virtualbox" do |vb|
      vb.name   = "hokage-dc01"
      vb.memory = 3072
      vb.cpus   = 2
    end
  end

  # --- Anbu-Srv01: IT server (IIS + file share), unconstrained Kerberos delegation ---
  config.vm.define "anbu-srv01" do |m|
    m.vm.box = "gusztavvargadr/windows-server-2022-standard"
    m.vm.hostname = "anbu-srv01"
    m.vm.network "private_network", ip: "192.168.56.11"
    m.vm.provider "virtualbox" do |vb|
      vb.name   = "anbu-srv01"
      vb.memory = 2560
      vb.cpus   = 2
    end
  end

  # --- Mission-Srv01: business server (simulated SQL/backup/applications) ---
  config.vm.define "mission-srv01" do |m|
    m.vm.box = "gusztavvargadr/windows-server-2022-standard"
    m.vm.hostname = "mission-srv01"
    m.vm.network "private_network", ip: "192.168.56.12"
    m.vm.provider "virtualbox" do |vb|
      vb.name   = "mission-srv01"
      vb.memory = 2560
      vb.cpus   = 2
    end
  end

  # --- Academy-WS01: client workstation, entry point of the attack chain ---
  config.vm.define "academy-ws01" do |m|
    m.vm.box = "gusztavvargadr/windows-10"
    m.vm.hostname = "academy-ws01"
    m.vm.network "private_network", ip: "192.168.56.13"
    m.vm.provider "virtualbox" do |vb|
      vb.name   = "academy-ws01"
      vb.memory = 2048
      vb.cpus   = 2
    end
  end
end
