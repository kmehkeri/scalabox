# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'yaml'
require './strip'

VAGRANT_ROOT = File.dirname(File.expand_path(__FILE__))
LOCAL_CONFIG_FILE = File.join(VAGRANT_ROOT, 'local.yml')
 
localconfig = File.exists?(LOCAL_CONFIG_FILE) ? YAML.load_file(File.join(VAGRANT_ROOT, 'local.yml')) : nil
proxy_enabled  = localconfig['proxy']['enabled']  rescue false
proxy_host     = localconfig['proxy']['host']     rescue ''
proxy_username = localconfig['proxy']['username'] rescue ''
proxy_password = localconfig['proxy']['password'] rescue ''
proxy_exclude  = localconfig['proxy']['exclude']  rescue ''

Vagrant.configure(2) do |config|
  # Base box
  config.vm.box = "chenhan/lubuntu-desktop-18.04"
  config.vm.box_check_update = false

  # Network proxy
  if Vagrant.has_plugin?("vagrant-proxyconf")
    if proxy_enabled
      config.proxy.http     = "http://#{proxy_username}:#{proxy_password}@#{proxy_host}"
      config.proxy.https    = "http://#{proxy_username}:#{proxy_password}@#{proxy_host}"
      config.proxy.no_proxy = "#{proxy_exclude}"
    else
      config.proxy.http     = ""
      config.proxy.https    = ""
      config.proxy.no_proxy = ""
    end
  end

  # Provider-specific configuration
  config.vm.provider "virtualbox" do |vb|
    vb.name = "scalabox"
    vb.cpus = 2
    vb.memory = "4096"
    vb.gui = true
    vb.customize ["modifyvm", :id, "--vram", "32"]
    vb.customize ["modifyvm", :id, "--usb", "off"]
    vb.customize ["modifyvm", :id, "--usbehci", "off"]
  end

  # System provisioning
  config.vm.provision "system", type: "shell", inline: <<-EOF.strip_heredoc
    # Add software sources
    apt-get -y update
    apt-get -y install apt-transport-https
    # * SBT
    echo "deb https://dl.bintray.com/sbt/debian /" >/etc/apt/sources.list.d/sbt.list
    apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 642AC823
    # * Docker
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

    # Install software from repositories
    apt-get -y update
    apt-get -y autoremove
    apt-get -y install docker-ce \
                       git \
                       guake \
                       httpie \
                       scala \
                       sbt \
                       vim
    
    # Install Ammonite
    sh -c '(echo "#!/usr/bin/env sh" && curl -L https://github.com/lihaoyi/Ammonite/releases/download/1.6.0/2.12-1.6.0) > /usr/local/bin/amm && chmod +x /usr/local/bin/amm'

    # Docker group for vagrant user
    usermod -aG docker vagrant

    # Autologin vagrant user
    cat <<-FOF >/etc/lightdm/lightdm.conf.d/99-autologin-vagrant.conf
    [Seat:*]
    autologin-user=vagrant
    FOF
  EOF

  # User provisioning
  config.vm.provision "user", type: "shell", privileged: false, inline: <<-EOF.strip_heredoc
    # Autostart Guake
    mkdir -p ~/.config/autostart
    cat <<-FOF >~/.config/autostart/guake.desktop
    [Desktop Entry] 
    Type=Application
    Exec=guake
    FOF

    # Install dotfiles
    git clone https://github.com/kmehkeri/dotfiles.git ~/dotfiles
    touch {.bashrc,.bash_profile,.vimrc,.gitconfig}
    ~/dotfiles/install.sh
    rm -f ~/{.bashrc.old,.bash_profile.old,.vimrc.old,.gitconfig.old}

    # Install VIM Scala plugin
    mkdir -p ~/.vim/{ftdetect,indent,syntax} && for d in ftdetect indent syntax ; do wget -nv -O ~/.vim/$d/scala.vim https://raw.githubusercontent.com/derekwyatt/vim-scala/master/$d/scala.vim; done

    # Git setup
    git config --global credential.helper "cache --timeout=86400"

    # Ammonite setup
    mkdir -p ~/.ammonite
    touch ~/.ammonite/session
  EOF

end