# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure(2) do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://atlas.hashicorp.com/search.
  # Disable automatic box update checking. If you disable this, then
  # boxes will only be checked for updates when the user runs
  # `vagrant box outdated`. This is not recommended.
  # config.vm.box_check_update = false

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.

  # Create a private network, which allows host-only access to the machine
  # using a specific IP.
  # config.vm.network "private_network", ip: "192.168.33.10"

  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.
  # config.vm.network "public_network"

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  # config.vm.synced_folder "../data", "/vagrant_data"

  # Provider-specific configuration so you can fine-tune various
  # backing providers for Vagrant. These expose provider-specific options.
  # Example for VirtualBox:
  #
  # config.vm.provider "virtualbox" do |vb|
  #   # Display the VirtualBox GUI when booting the machine
  #   vb.gui = true
  #
  #   # Customize the amount of memory on the VM:
  #   vb.memory = "1024"
  # end
  #
  # View the documentation for the provider you are using for more
  # information on available options.

  # Define a Vagrant Push strategy for pushing to Atlas. Other push strategies
  # such as FTP and Heroku are also available. See the documentation at
  # https://docs.vagrantup.com/v2/push/atlas.html for more information.
  # config.push.define "atlas" do |push|
  #   push.app = "YOUR_ATLAS_USERNAME/YOUR_APPLICATION_NAME"
  # end

  # Enable provisioning with a shell script. Additional provisioners such as
  # Puppet, Chef, Ansible, Salt, and Docker are also available. Please see the
  # documentation for more information about their specific syntax and use.
  config.vm.define "prom" do |prom|
    prom.vm.box = "ubuntu/xenial64"
    prom.vm.network "private_network", ip: "192.168.33.10"
    prom.vm.network "forwarded_port", guest: 9090, host: 9090
    prom.vm.provision "docker"
    prom.vm.provision "shell", inline: <<-SHELL
    docker rm -f prom
    docker rm -f pg_prometheus
    docker rm -f prometheus_postgresql_adapter
    docker run --name pg_prometheus -d -p 5432:5432 timescale/pg_prometheus:master postgres \
      -csynchronous_commit=off
    sleep 5 # wait for postgres
    docker run --name prometheus_postgresql_adapter --link pg_prometheus -d -p 9201:9201 \
      timescale/prometheus-postgresql-adapter:master \
      -pg.host=pg_prometheus \
      -pg.prometheus-log-samples
    sleep 5 # wait for postgres adapter
    docker run --link prometheus_postgresql_adapter --name prom -d -p 9090:9090 \
      -v /vagrant/prometheus.yml:/etc/prometheus/prometheus.yml \
      -v /vagrant/rules.yml:/etc/prometheus/rules.yml \
      prom/prometheus --web.enable-lifecycle --config.file /etc/prometheus/prometheus.yml
    SHELL
  end
  config.vm.define "node1" do |n1|
    n1.vm.network "private_network", ip: "192.168.33.11"
    n1.vm.network "forwarded_port", guest: 9100, host: 9100
    n1.vm.box = "ubuntu/xenial64"
    n1.vm.provision "shell", inline: <<-SHELL
    mkdir -p /usr/local/node_exporter/textfiles
    cd /usr/local/node_exporter
    wget https://github.com/prometheus/node_exporter/releases/download/v0.15.2/node_exporter-0.15.2.linux-amd64.tar.gz
    tar xvfz node_exporter-0.15.2.linux-amd64.tar.gz
    ln -s node_exporter-0.15.2.linux-amd64 current
    useradd node_exporter -s /sbin/nologin
    cat <<EOF >/etc/default/node_exporter
export OPTIONS="--collector.textfile.directory /usr/local/node_exporter/textfiles"
EOF
    cat <<EOF >/etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter

[Service]
User=node_exporter
EnvironmentFile=/etc/default/node_exporter
ExecStart=/usr/local/node_exporter/current/node_exporter $OPTIONS

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable node_exporter
systemctl start node_exporter
SHELL
  end
  config.vm.define "node_random" do |nr|
    nr.vm.network "private_network", ip: "192.168.33.13"
    nr.vm.network "forwarded_port", guest: 8081, host: 8081
    nr.vm.network "forwarded_port", guest: 8082, host: 8082
    nr.vm.network "forwarded_port", guest: 8083, host: 8083
    nr.vm.box = "ubuntu/xenial64"
    nr.vm.provision "shell", inline: <<-SHELL
    apt-get update
    apt-get install -y git golang-go
    mkdir -p /usr/local/random_client
    cd /usr/local/random_client
    if [[ ! -d client_golang ]] ; then
      git clone https://github.com/prometheus/client_golang.git
    fi
    export GOPATH=/root/go
    cd client_golang/examples/random
    go get -d
    go build

    for i in 1 2 3 ; do
      cat <<EOF >/etc/systemd/system/random_client_$i.service
[Unit]
Description=Random Client $i

[Service]
ExecStart=/usr/local/random_client/client_golang/examples/random/random -listen-address=:808${i}

[Install]
WantedBy=multi-user.target
EOF
    systemctl daemon-reload
    systemctl enable random_client_$i
    systemctl start random_client_$i
    done
SHELL
  end
  config.vm.define "grafana" do |g|
    g.vm.network "private_network", ip: "192.168.33.12"
    g.vm.network "forwarded_port", guest: 8000, host: 8000
    g.vm.box = "ubuntu/xenial64"
    g.vm.provision "shell", inline: <<-SHELL
    cat >/etc/apt/sources.list.d/grafana.list <<APT
deb https://packagecloud.io/grafana/stable/debian/ stretch main
APT
curl https://packagecloud.io/gpg.key | apt-key add -
    apt-get install -y apt-transport-https
    apt-get update
    apt-get install -y adduser libfontconfig
    apt-get install -y grafana
SHELL
  end
end
