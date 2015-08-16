#Use Raspberry Pi as Backup and NAS using ssfs

###Connect Raspberry Pi with WiFi to internet

connect to your pi e.g. over ssh using pi@ip.of.pi and raspberry as password.

change /etc/network/interfaces
e.g. by `sudo vi /etc/network/interfaces` to:
```
auto lo
iface lo inet loopback
iface eth0 inet dhcp

auto wlan0
allow-hotplug wlan0
iface wlan0 inet dhcp
wpa-ap-scan 1
wpa-scan-ssid 1
wpa-ssid "DEIN-WLAN-NAME"
wpa-psk "DEIN-WLAN-SCHLÜSSEL"
```

###Use reverse ssh and small server to make it available from the internet

How does this work?

The Raspberry will connect with a reverse ssh to the server S. Then the client will also connect to the server. Then, once on the server, the client cann connect to the Raspberry:

Client --> Server <-- Raspberry

For further information, e..g check [Stackoverflow](http://unix.stackexchange.com/questions/76935/how-can-i-mount-a-remote-sshfs-directory-of-an-publicly-inaccessible-server-on-m)

On raspberry, we will use autossh to re-establish the ssh connection all the time.

`sudo apt-get install autossh`

generate the keys (press enter all the time for no keyfile password) and do a key exchange with the server:
```
ssh-keygen -t rsa
ssh-copy-id pi@server.com
```

then run `sudo vi /etc/rc.local` and add the following to run the command at startup. This will establish a reverse ssh connection on port 19001 on the server and use 20001 as the status ports
```
sudo -u pi /usr/bin/autossh \
  -M 19001 -o ServerAliveInterval=5 -o ServerAliveCountMax=1 -R 18001:localhost:22 -N pi@server.org &
```

Now, on the **client**, do the following:

`sudo apt-get install autossh`

do the key exchange again:

```
ssh-keygen -t rsa
ssh-copy-id pi@server.com
```

Now you can forward the exposed port of the pi on the server to your local machine using `ssh -L 18001:localhost:18001 -N pi@server.org`

and finally connect to your raspberry from your client using `ssh -p 18001 pi@localhost`

To automatically forward the ssh connection, one can use autossh at startup. Run `sudo vi /etc/rc.local` on your client and add the following to run the command at startup.
```
#connect to the server and establish a port forward to our raspberry server
#which is connected to the server over a reverse ssh connection
sudo -u main /usr/bin/autossh \
  -M 19005 -o ServerAliveInterval=5 -o ServerAliveCountMax=-L 18001:localhost:18001 -N pi@server.org &

```

and to automatically mount the filesystem of the raspberry to your client, you can use sshfs
```
sudo apt-get install sshfs
mkdir ~/nas
```

To check the connection, you can run `sshfs -p 18001 pi@localhost:/home/pi ~/nas/ -o reconnect -o ServerAliveInterval=5`

#####TODO: Write how to add it automatically on startup
or add it to the fstab file to run it automatically at startup:
edit /etc/fstab and add:
```
sshfs#pi@localhost:/home/pi /media/nas fuse port=18001,noauto,user 0 0

```

###Issues and system freeze when opening folder or trying to read it
This might happen because the network connection or the ssh connection is timing out, but taking some time to do so.
One "workaround" is to run `sudo umount ~/nas -f`. Check also this [ubuntu bug](https://bugs.launchpad.net/ubuntu/+source/sshfs-fuse/+bug/159031).
