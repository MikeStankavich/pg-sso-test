# -*- mode: ruby -*-
# vi: set ft=ruby :

ENV['VAGRANT_DEFAULT_PROVIDER'] = 'virtualbox'

box = {
        :name => "ssou1",
        :hostname => "box2",
        :domain => "myrealm.example",
        :realm => "MYREALM.EXAMPLE",
        :private_ip => "192.168.215.11",
        :mem => "512",
        :cpu => "2",
        :kdb_pass => "foobar"
    }


Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/trusty64"

  config.vm.provider "virtualbox" do |vb|
    # vb.gui = true
    # Customize the amount of memory on the VM:
    vb.memory = box[:mem]
    # vb.cpu = box[:cpu]
  end

  # Create a private network, which allows host-only access to the machine
  # using a specific IP.
  config.vm.network "private_network", ip: box[:private_ip]


  config.vm.provision 'file', source: './krb5.conf', destination: './krb5.conf'
  config.vm.provision "shell", privileged: true, \
    path: "https://anonscm.debian.org/cgit/pkg-postgresql/postgresql-common.git/plain/pgdg/apt.postgresql.org.sh"

#  apt-get -y install krb5-admin-server krb5-kdc krb5-config krb5-user
  config.vm.provision "shell", privileged: true, inline: <<-SHELL
    echo Fix stdin is not a tty error...
    sed -i '/tty/!s/mesg n/tty -s \\&\\& mesg n/' /root/.profile

    echo Update package manager metadata...
    DEBIAN_FRONTEND=noninteractive
    apt-get update

    echo Install Kerberos packages...
    echo "krb5-config krb5-config/default_realm string #{box[:realm]}" | debconf-set-selections
    echo "krb5-admin-server krb5-admin-server/kadmind boolean true" | debconf-set-selections
    DEBIAN_FRONTEND=noninteractive apt-get -y install krb5-kdc krb5-user krb5-admin-server

    echo Add private ip to /etc/hosts...
    grep -q "#{box[:private_ip]} #{box[:domain]}" /etc/hosts || \
      sed -i "/127\\.0\\.0\\.1\\s\\+localhost/a #{box[:private_ip]} \
      #{box[:domain]} #{box[:hostname]}.#{box[:domain]}" /etc/hosts

    echo Copy krb5 config file into /etc/...
    cp /home/vagrant/krb5.conf /etc/

    echo Create the KDC database...
    kdb5_util create -s -W -P "#{box[:kdb_pass]} #{box[:kdb_pass]}"

    echo Add admin principal...
    kadmin.local -q "addprinc -pw #{box[:kdb_pass]} postgres/admin"

    echo Create the KDC daemon...
    krb5kdc

    echo Add postgres principal...
    kadmin.local -q "addprinc -pw #{box[:kdb_pass]} postgres/myrealm.example@MYREALM.EXAMPLE"

    echo Create keytab file...
    kadmin.local -q "xst -k myrealm.example.keytab postgres/myrealm.example@MYREALM.EXAMPLE"

    echo Install Postgresql server...
    apt-get -y install postgresql-9.5

    echo Update Postgres config files...
    sed -i "s/#listen_addresses = 'localhost'/listen_addresses = '*'/" /etc/postgresql/9.5/main/postgresql.conf
    sed -i "s|#krb_server_keyfile = ''|krb_server_keyfile = '/etc/postgresql/9.5/main/postgres.keytab'|" \
      /etc/postgresql/9.5/main/postgresql.conf
    grep -q krb_server_keyfile /etc/postgresql/9.5/main/postgresql.conf || \
      echo -e "\n# support AD auth\nkrb_server_keyfile = '/etc/postgresql/9.5/main/postgres.keytab'" >> \
      /etc/postgresql/9.5/main/postgresql.conf
    grep -q "host.*gss" /etc/postgresql/9.5/main/pg_hba.conf || \
      echo -e "\n# support AD auth\n host    \
      all       all     0.0.0.0/0    gss include_realm=1 krb_realm=#{box[:realm]}" >> /etc/postgresql/9.5/main/pg_hba.conf
    cp myrealm.example.keytab /etc/postgresql/9.5/main/postgres.keytab
    chown postgres:postgres /etc/postgresql/9.5/main/postgres.keytab

    echo Start Postgres service...
    service postgresql restart

    echo Test Postgres Kerberos Auth...
    sudo -u postgres psql -c 'CREATE ROLE "postgres/myrealm.example@MYREALM.EXAMPLE" SUPERUSER LOGIN'
    kinit -k -t myrealm.example.keytab  postgres/myrealm.example@MYREALM.EXAMPLE -V
    klist
    psql -U "postgres/myrealm.example@MYREALM.EXAMPLE" -h myrealm.example postgres -c "\\dg+"
  SHELL

end

# for ubuntu:  apt-get install krb5-admin-server, krb5-kdc, krb5-config, krb5-user, krb5-clients, and krb5-rsh-server