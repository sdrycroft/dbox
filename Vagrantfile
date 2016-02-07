require 'yaml'
settings = YAML.load_file 'config.yml'
#
# README
# ======
#
# 1. Install nfs packages on the host system
#
#    $ sudo apt-get install nfs-kernel-server
#
# 2. Sign up for NordVPN and add your username/password to the configuration
#    above. Note, due to BASH substitution avoid using $ or !.
#
# 3. Ensure the user running vagrant has full sudo privileges. There is no need
#    for "sudo" to be passwordless, it just means you'll be asked for your
#    password during "vagrant up".
#
#-------------------------------------------------------------------------------
$bootstrap = "#!/bin/bash

#-- APT PACKAGES ---------------------------------------------------------------

cat << EOF > /etc/apt/sources.list.d/ffmpeg.list
deb http://www.deb-multimedia.org jessie main non-free
deb-src http://www.deb-multimedia.org jessie main non-free
EOF
cat << EOF > /etc/apt/sources.list.d/sonarr.list
deb http://apt.sonarr.tv/ master main
EOF
cat << EOF > /etc/apt/sources.list.d/mono-xamarin.list
deb http://download.mono-project.com/repo/debian wheezy-apache24-compat main
deb http://download.mono-project.com/repo/debian wheezy main
deb http://download.mono-project.com/repo/debian wheezy-libjpeg62-compat main
EOF
apt-get update
apt-get install deb-multimedia-keyring -y --force-yes
apt-key adv --keyserver keyserver.ubuntu.com --recv-keys FDA5DFFC
apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 3FA7E0328081BFF6A14DA29AA6A19B38D3D831EF
apt-get update
apt-get upgrade
apt-get install -y \
	apache2 \
	curl \
	git \
	libapache2-mod-php5 \
	mediainfo \
	nzbdrone \
	openvpn \
	rtorrent \
	screen \
	unrar-free \
	unzip \
	ffmpeg

#-- APACHE2 CONFIGURATION ------------------------------------------------------

a2enmod proxy_http
a2enmod rewrite
cat << EOF > /etc/apache2/sites-available/000-default.conf
<VirtualHost *:80>
        DocumentRoot /var/www/html
        ProxyPass /couchpotato/ http://127.0.0.1:5050/couchpotato/
        ProxyPassReverse /couchpotato/ http://127.0.0.1:5050/couchpotato/
        ProxyPass /nzbdrone/ http://127.0.0.1:8989/nzbdrone/
        ProxyPassReverse /nzbdrone/ http://127.0.0.1:8989/nzbdrone/
        RewriteEngine on
        RewriteCond %{REQUEST_URI} ^/couchpotato$
        RewriteRule .* /couchpotato/ [L,R]
        RewriteCond %{REQUEST_URI} ^/nzbdrone$
        RewriteRule .* /nzbdrone/ [L,R]
</VirtualHost>
EOF

#-- RUTORRENT ------------------------------------------------------------------

# Clone ruTorrent
rm /var/www/html/index.html
git clone https://github.com/Novik/ruTorrent.git /var/www/html

# Copy the default configuration file if we have not already got one
if [ ! -f /home/vagrant/.rtorrent.rc ]
then
	cp /home/vagrant/.rtorrent.rc.default /home/vagrant/.rtorrent.rc
fi
chown -R vagrant:vagrant /home/vagrant/.rtorrent.rc

#-- NZBDRONE ---------------------------------------------------------------------

# Create the user to run nzbdrone as
useradd -d /opt/NzbDrone -r -s /bin/sh nzbdrone

cat << EOF > /etc/systemd/system/nzbdrone.service
[Unit]
Description=NZBDrone
After=network.target

[Service]
User=nzbdrone
ExecStart=/usr/bin/mono /opt/NzbDrone/NzbDrone.exe

[Install]
WantedBy=multi-user.target
EOF

#-- COUCHPOTATO ----------------------------------------------------------------

# Clone CouchPotato
git clone https://github.com/RuudBurger/CouchPotatoServer.git /usr/local/share/couchpotato
cd /usr/local/share/couchpotato
git checkout build/3.0.1

# Create the user to run couchpotato as
useradd -d /home/couchpotato -r -s /bin/sh couchpotato

cat << EOF > /etc/systemd/system/couchpotato.service
[Unit]
Description=CouchPotato
After=network.target

[Service]
User=couchpotato
ExecStart=/usr/bin/python /usr/local/share/couchpotato/CouchPotato.py

[Install]
WantedBy=multi-user.target
EOF

# Copy the default configuration file if we have not already got one
if [ ! -f /home/couchpotato/.couchpotato/settings.conf ]
then
	cp /home/couchpotato/.couchpotato/settings.conf.default /home/couchpotato/.couchpotato/settings.conf
fi
chown -R couchpotato:couchpotato /home/couchpotato

#-- NORDVPN --------------------------------------------------------------------

# Download the nordvpn settings
mkdir /etc/openvpn/nordvpn
cd /etc/openvpn/nordvpn
wget --quiet https://nordvpn.com/api/files/zip
unzip -qq zip
rm zip
cd ..
ln -s nordvpn/uk20.nordvpn.com.udp1194.ovpn nordvpn.conf 

# Update the configuration files so that they get the username/password from a file.
cd /etc/openvpn/nordvpn
for i in $(ls -1)
do
	sed 's|auth-user-pass\s*$|auth-user-pass /etc/openvpn/nordvpn/pass.txt|' $i -i
done

# Add the username/password
cat << EOF > /etc/openvpn/nordvpn/pass.txt
#{settings['nordvpn']['username']}
#{settings['nordvpn']['password']}
EOF

cat << EOF > /etc/systemd/system/nordvpn.service
[Unit]
Description=NordVPN service
After=network.target

[Service]
ExecStart=/usr/sbin/openvpn /etc/openvpn/nordvpn.conf

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload

#-- IPTABLES -------------------------------------------------------------------

iptables -A OUTPUT -o tun0 -j ACCEPT
iptables -A OUTPUT -o tun1 -j ACCEPT
iptables -A OUTPUT -p tcp -d 192.168.0.0/16 -j ACCEPT
iptables -A OUTPUT -p tcp -d 127.0.0.0/16 -j ACCEPT
iptables -A OUTPUT -p tcp -d 10.0.0.0/16 -j ACCEPT
iptables -A OUTPUT -p tcp -j DROP

#-- START SERVICES -------------------------------------------------------------

sleep 30

# Start services
service openvpn stop
service nordvpn start
service couchpotato start
service nzbdrone start
service apache2 restart
# Start rtorrent
su vagrant -l -c 'screen -d -m rtorrent'
chown www-data:www-data -R /var/www/html";

#-- VAGRANT CONFIGURATION ------------------------------------------------------
Vagrant.configure(2) do |config|
  config.vm.box = "debian/jessie64"
  config.vm.network "public_network", ip: settings['ips']['public']
  config.vm.network "private_network", ip: settings['ips']['private']
  config.vm.synced_folder "vagrant-home", "/home/vagrant", type: "nfs"
  config.vm.synced_folder "couchpotato-home", "/home/couchpotato", type: "nfs"
  config.vm.synced_folder "nzbdrone-home", "/home/nzbdrone", type: "nfs"
  config.vm.synced_folder ".", "/vagrant", disabled: true
  config.vm.provider "virtualbox" do |vb|
    vb.memory = settings['memory']
    vb.name = "download"
  end
  config.vm.provision "shell", inline: $bootstrap
end
