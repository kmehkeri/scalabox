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

APT_UPGRADE = false
SPARK_VERSION = "2.4.0"
INTELLIJ_VERSION = "IC-2018.3.2"

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
    vb.customize ["modifyvm", :id, "--clipboard", "bidirectional"]
  end

  # System provisioning
  config.vm.provision "system-software", type: "shell", inline: <<-EOF.strip_heredoc
    echo "=== Add software sources ==="
    export DEBIAN_FRONTEND=noninteractive
    apt-get -y update
    apt-get -y install apt-transport-https

    echo "=== ... for SBT ==="
    echo "deb https://dl.bintray.com/sbt/debian /" >/etc/apt/sources.list.d/sbt.list
    apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 642AC823
    
    echo "=== ... for Docker ==="
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
    add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

    echo "=== Install/remove software ==="
    apt-get -y update
    apt-get -y remove light-locker
    apt-get -y autoremove
    #{APT_UPGRADE ? "apt-get -y upgrade" : ""}
    apt-get -y install docker-ce \
                       git \
                       gitg \
                       guake \
                       httpie \
                       openjdk-8-jdk-headless \
                       scala \
                       sbt \
                       vim
  EOF

  config.vm.provision "system-docker-compose", type: "shell", inline: <<-EOF.strip_heredoc
    echo "=== Install Docker-compose ==="
    curl -fLsS "https://github.com/docker/compose/releases/download/1.23.2/docker-compose-Linux-x86_64" -o /usr/local/bin/docker-compose
    chmod +x /usr/local/bin/docker-compose
  EOF

  config.vm.provision "system-ammonite", type: "shell", inline: <<-EOF.strip_heredoc
    echo "=== Install Ammonite ==="
    sh -c '(echo "#!/usr/bin/env sh" && curl -fLsS https://github.com/lihaoyi/Ammonite/releases/download/1.6.0/2.12-1.6.0) > /usr/local/bin/amm && chmod +x /usr/local/bin/amm'
  EOF

  config.vm.provision "system-spark", type: "shell", inline: <<-EOF.strip_heredoc
    echo "=== Install Spark ==="
    spark_file=spark-#{SPARK_VERSION}-bin-hadoop2.7
    wget -nv https://archive.apache.org/dist/spark/spark-#{SPARK_VERSION}/${spark_file}.tgz
    tar -xzf ${spark_file}.tgz
    rm ${spark_file}.tgz
    mv ${spark_file} /opt/
    ln -s /opt/${spark_file} /opt/spark

    echo "=== Environment variables for Spark ==="
    cat <<-FOF >/etc/profile.d/spark.sh
    export SPARK_HOME=/opt/spark
    export PATH=\\${PATH}:\\${SPARK_HOME}/bin
    FOF
  EOF

  config.vm.provision "system-intellij", type: "shell", inline: <<-EOF.strip_heredoc
    echo "=== Install IntelliJ ==="
    wget -nv https://download.jetbrains.com/idea/idea#{INTELLIJ_VERSION}.tar.gz
    tar -xzf idea#{INTELLIJ_VERSION}.tar.gz
    ideadir=$(tar -tzf idea#{INTELLIJ_VERSION}.tar.gz | head -1 | cut -d'/' -f1)
    rm idea#{INTELLIJ_VERSION}.tar.gz
    mv ${ideadir} /opt
    ln -s /opt/${ideadir} /opt/idea
  EOF

  config.vm.provision "system-other", type: "shell", inline: <<-EOF.strip_heredoc
    echo "=== Autologin vagrant user ==="
    cat <<-FOF >/etc/lightdm/lightdm.conf.d/99-autologin-vagrant.conf
    [Seat:*]
    autologin-user=vagrant
    FOF
    
    echo "=== Add vagrant to docker group ==="
    usermod -aG docker vagrant
  EOF

  # User provisioning
  config.vm.provision "user", type: "shell", privileged: false, inline: <<-EOF.strip_heredoc
    echo "=== Autostart Guake ==="
    mkdir -p ~/.config/autostart
    cat <<-FOF >~/.config/autostart/guake.desktop
    [Desktop Entry] 
    Type=Application
    Exec=guake
    FOF

    echo "=== Install dotfiles ==="
    git clone https://github.com/kmehkeri/dotfiles.git ~/dotfiles
    touch {.bashrc,.bash_profile,.vimrc,.gitconfig}
    ~/dotfiles/install.sh
    rm -f ~/{.bashrc.old,.bash_profile.old,.vimrc.old,.gitconfig.old}

    echo "=== Install Scala plugin for VIM ==="
    mkdir -p ~/.vim/{ftdetect,indent,syntax} && for d in ftdetect indent syntax ; do wget -nv -O ~/.vim/$d/scala.vim https://raw.githubusercontent.com/derekwyatt/vim-scala/master/$d/scala.vim; done

    echo "=== Git config ==="
    git config --global credential.helper "cache --timeout=86400"

    echo "=== Ammonite setup ==="
    mkdir -p ~/.ammonite
    touch ~/.ammonite/session

    echo "=== IntelliJ launcher ==="
    mkdir -p ~/Desktop
    cat <<-FOF >~/Desktop/intellij.desktop
    #!/usr/bin/env xdg-open
    [Desktop Entry]
    Version=1.0
    Type=Application
    Terminal=false
    Exec=/opt/idea/bin/idea.sh
    Name=IntelliJ Idea
    Comment=IntelliJ Idea
    Icon=/opt/idea/bin/idea.png
    FOF
  EOF

end
