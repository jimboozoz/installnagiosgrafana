#Guide to install Nagios, InfluxDB and Grafana for analysis.

##Getting Started

###Nagios, Nagios Plugins and Apache Installation.

Install the gcc compiler, build-essentials and apache2.

```
sudo apt-get install wget build-essential apache2 php apache2-mod-php7.0 php-gd libgd-dev sendmail unzip
```

Create a new user and group for Nagios.

```
useradd Nagios

groupadd nagcmd

usermod -a -G nagcmd Nagios

usermod -a -G nagios,nagcmd www-data
```

Download and extract Nagios core.

```
cd ~

wget https://assets.nagios.com/downloads/nagioscore/releases/nagios-4.2.0.tar.gz

tar -xzf nagios*.tar.gz

cd nagios-4.2.0
```

Configure Nagios with the user and group.

```
./configure --with-nagios-group=nagios --with-command-group=nagcmd
```

Install Nagios.

```
make all

sudo make install

sudo make install-commandmode

sudo make install-init

sudo make install-config

/usr/bin/install -c -m 644 sample-config/httpd.conf /etc/apache2/sites-available/nagios.conf
Copy evenhandler directory to Nagios directory.
cp -R contrib/eventhandlers/ /usr/local/nagios/libexec/

chown -R nagios:nagios /usr/local/nagios/libexec/eventhandlers
Download Nagios Plugins.
cd ~

wget https://nagios-plugins.org/download/nagios-plugins-2.1.2.tar.gz

tar -xzf nagios-plugins*.tar.gz

cd nagios-plugin-2.1.2/
```

Install Nagios Plugins.

```
./configure --with-nagios-user=nagios --with-nagios-group=nagios --with-openssl

make

make install

```
Nagios will be installed in /usr/local/Nagios/ now configure the Nagios Contact.

```
vim /usr/local/nagios/etc/nagios.cfg
```

Uncomment the line 51 for the host monitor.

```
cfg_dir=/usr/local/nagios/etc/servers

```
Add a new folder named “servers”.

```
mkdir -p /usr/local/nagios/etc/servers
```

Configure the Nagios contact, opening the contacts.cfg and using you email instead of the example.

```
vim /usr/local/nagios/etc/objects/contacts.cfg
```

Enable Apache modules.

```
sudo a2enmod rewrite

sudo a2enmod cgi
```

Configure a nagiosadmin for the Nagios web interface.

```
sudo htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin

```
Enable the Nagios Virtualhost.

```
sudo ln -s /etc/apache2/sites-available/nagios.conf /etc/apache2/sites-enabled/
```

For the Nagios initialization,copy the next file.

```
cd /etc/init.d/

cp /etc/init.d/skeleton /etc/init.d/nagios
```

Edit the file, adding the next commands at the end.

```
DESC="Nagios"
NAME=nagios
DAEMON=/usr/local/nagios/bin/$NAME
DAEMON_ARGS="-d /usr/local/nagios/etc/nagios.cfg"
PIDFILE=/usr/local/nagios/var/$NAME.lock
Make it executable and run Apache and Nagios.
chmod +x /etc/init.d/nagios
```

Restart Apache and Nagios

```
sudo service apache2 start

sudo service nagios start
```

Now you can access Nagios from “localhost/Nagios/”.

#InfluxDB Installation.


Upgrade and update Ubuntu.

```
sudo apt-get update

sudo apt-get upgrade
```

Add the InfluxData repository with the next three commands, this will create a file called “/etc/apt/sources.list.d/influxdb.list”

```
curl -sL https://repos.influxdata.com/influxdb.key | sudo apt-key add -

source /etc/lsb-release

echo "deb https://repos.influxdata.com/${DISTRIB_ID,,} ${DISTRIB_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
```

Install InfluxDB.

```
sudo apt-get update && sudo apt-get install influxdb
```

Start InfluxDB.

```
sudo service influxdb start
```

Connect to the influxDB with the command “influx”, and create a new Database called Nagios.

```
Influx
CREATE DATABASE Nagios
```

For closing influx type “exit” or “quit”.

Grafana Installation.

Install the Stable version.

```
wget https://s3-us-west-2.amazonaws.com/grafana-releases/release/grafana_4.5.2_amd64.deb
sudo apt-get install -y adduser libfontconfig
sudo dpkg -i grafana_4.5.2_amd64.deb
```

Edit the next file.

```
vim /etc/apt/sources.list
```

Add the following line at the end of “sources.list”

```
deb https://packagecloud.io/grafana/stable/debian/ jessie main
```

Add the PackageCloud key for installing the packages.

```
curl https://packagecloud.io/gpg.key | sudo apt-key add -
```

Update the repositories and install grafana.

```
sudo apt-get update
sudo apt-get install grafana
```

To fetch packages over HTTPS, now use the next command

```
sudo apt-get install –y apt-transport-https
```

To start Grafana use the next command

```
sudo service grafana-server start
```

Now you can access Grafana from “http://localhost:3000”.

Nagios2influx plugin Installation.
For this steps you need to have installed git in your Ubuntu Console, with your name, email and the SSH key to your github user. You can follow the next instructions https://www.digitalocean.com/community/tutorials/how-to-install-git-on-ubuntu-14-04.
Clone the next repository. https://github.com/PickinA/nagios2influx

```
git clone git@github.com:PickinA/nagios2influx.git
```

Build and install the repository.

```
cd nagios2influx
make rpm
sudo make install
```

Validate that the next files were created with the same permissions and users, if not do it manually using the cloned folder.

```
-rw-rw----    1 nagios  nagios /etc/nagios/nagios2influx.cfg
-rwxr-xr-x    1 root    root   /usr/bin/nagios-perf
-rwxr-xr-x    1 root    root   /usr/bin/nagios2influx
-rw-r--r--    1 root    root   /usr/share/man/man1/nagios2influx.1.gz
-rw-r--r--    1 root    root   /usr/share/man/man5/nagios2influx.cfg.5.gz
```

Edit Nagios.cfg with the next command.

```
nano /usr/local/nagios/etc/nagios.cfg
```

Inside the file, modify process_performance_data=0 to process_performance_data=1,
Uncomment service_perfdata_command=process-service-perfdata,

Modify and uncomment service_perfdata_file_processing_interval=0 to service_perfdata_file_processing_interval=120 (or any number of seconds, in this case Nagios will print to the database the data every 120 seconds)

And uncomment service_perfdata_file_processing_command=process-service-perfdata-file

Edit commands.cfg with the next command.

```
nano /usr/local/nagios/etc/objects/commands.cfg
```

At the end to the file add the next command.

```
#Command for Nagios to influx #
define command{
		command_name            process-service-perfdata-file
		command_line            /usr/bin/nagios-perf > /tmp/nagios-perf 2>&1
		}
```

Modify nagios2influx.cfg file.

```
vim /etc/nagios/nagios2influx.cfg
```

Inside the file, change perfdata=/var/nagios/service-perfdata.out to perfdata=/usr/local/nagios/var/service-perfdata.out
Restart again apache2 and Nagios, the process will be working.

```
sudo service apache2 start

sudo service nagios start
```
