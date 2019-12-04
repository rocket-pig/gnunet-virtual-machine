# gnunet-virtual-machine
A fully working GNUNet virtual machine on Ubuntu 19.04 Server.  Both the OVA is available (*note: could be available, if anyone shows interest*) for download as are step-by-step instructions to do it yourself. 

# GNUNet Virtual Machine
# [Part 1] Ubuntu virtual machine:
### make our working dir:

    mkdir /opt/u64

### get the iso here:

    wget "http://releases.ubuntu.com/19.04/ubuntu-19.04-live-server-amd64.iso" -O /opt/u64/u64.iso

### build the vm:
You can totally do this the 'normal' way via the virtualbox GUI also.  I like command line so I learned  and created this way.  It's tried and tested and I use it to make 'burner' vms.

    export VM_BASEDIR="/opt/u64"
    export VM="u64"
    
    sudo mkdir $VM_BASEDIR
    sudo chmod a+rw $VM_BASEDIR
   
    #create the vm and the hd to attach. note you can set size of both hd/ram here. The defaults i set work fine.
    VBoxManage createvm --name $VM --register --basefolder $VM_BASEDIR
    VBoxManage createhd --filename $VM_BASEDIR/$VM.vdi --size 25000
    VBoxManage modifyvm $VM --memory 2048 --ostype "Ubuntu (64-bit)" \
        --vram 16 --pae off --x2apic on
    
    #create the SATA and IDE interfaces. Attach ISO and VDI to them.
    VBoxManage storagectl $VM --name "SATA" --add sata --controller IntelAHCI
    VBoxManage storageattach $VM --storagectl "SATA" --port 0 --device 0 --type hdd --medium $VM_BASEDIR/$VM.vdi
    VBoxManage storagectl $VM --name "IDE" --add ide
    VBoxManage storageattach $VM --storagectl "IDE" --port 0 \
     --device 0 --type dvddrive --medium $VM_BASEDIR/u64.iso
    
    #create NAT network, turn it on, connect our vm to it.
    VBoxManage natnetwork add --netname natnet1 --network "192.168.15.0/24" --enable
    VBoxManage natnetwork modify --netname natnet1 --dhcp on
    VBoxManage natnetwork start --netname natnet1
    VBoxManage modifyvm $VM --nic1 natnetwork --nictype1 82540EM --cableconnected1 on --nat-network1 natnet1
    
    #set up 'host only' interface on host, then set this VM to have an adapter connected to that net.
    VBoxManage hostonlyif
    VBoxManage modifyvm $VM --nic2 hostonly --hostonlyadapter2 vboxnet0
    
    VBoxManage startvm $VM

### install Ubuntu to virtual hard disk
Unfortunately I could not script this part.  I definitely tried, but the 'unattended install' parts of virtualbox are still in the works. 
There's nothing particular to GNUNet to do here, just click your way thru the usual timezone and username settings, they aren't important.  **Do Not pick 'gnunet' as the default username** though, as we will use that username in next section.
Once install is complete and you can successfully boot into it via GUI virtualbox, I like to ssh to it instead (copy-paste works, for starters):

### ssh into your new vbox:

    arp -a -i vboxnet0
...will get you your vbox's IP address if you dont know what it is.
copying your *~/.ssh/id_rsa.pub* into the vm's *~/.ssh/authorized_keys* (make it if it doesn't yet exist) is also nice for passwordless login.

You can also start the vm 'headless' from command line so you dont have to have virtualbox GUI open anymore.  With vm shut down, try:

    VBoxManage list vms
(it's called *u64* i think, unless you changed it). Start with:

    VBoxManage startvm $VM --type headless
..*$VM* being your vm name.  You can shut it down with:

    VBoxManage controlvm $VM poweroff


# [Part 2] GNUNet compile and install section:
Its important to note here that though these instructions were found piecemeal other places (official documentation mostly) - they Are Not an exact match and the instructions I found didn't work to begin with.  For example nothing worked initially because _libtool_ is now split into two packages on Ubuntu, _libtool_ and _libtool-bin_ and the errors were less than lucid about what was wrong.  TL;DR these are _updated_ instructions that i found elsewhere and initially didn't work and I had the joy of figuring out why everything was screaming error and segfaulting and so on for days so that I could compile this.

#### First we install required stuff:

    sudo apt install git libtool libtool-bin autoconf autopoint \
    build-essential libgcrypt-dev libidn11-dev zlib1g-dev \
    libunistring-dev libglpk-dev miniupnpc libextractor-dev \
    libjansson-dev libcurl4-gnutls-dev gnutls-bin mysql-server libsqlite3-dev \
    openssl libnss3-tools libmicrohttpd-dev libopus-dev libpulse-dev \
    libogg-dev

#### Now to git gnunet and start build:

    mkdir ~/gnunet_installation
    cd ~/gnunet_installation
    git clone --depth 1 https://gnunet.org/git/gnunet.git
    cd ~/gnunet_installation/gnunet
    export GNUNET_PREFIX=/usr/local 
    ./bootstrap

#ubuntu 19.04 apt is one microhttpd revision away. sry not sry

    sed -iv 's/0.9.63/0.9.62/g' configure
    sed -iv 's/0.9.63/0.9.62/g' configure.ac

#### Configure,make,make install time:

    ./configure --prefix=$GNUNET_PREFIX --disable-documentation 
    sudo addgroup gnunetdns
    sudo adduser --system --group --disabled-login --home /var/lib/gnunet gnunet
    sudo usermod -aG gnunet $USER
    make
    #make ends with '...leaving dir [etcetc]' and so we cd back in before install
    cd ~/gnunet_installation/gnunet
    sudo make install
    
    libtool --finish /lib

## Set up gnunet systemd service:
If a file requires creation/editing, I put them here as **echo|tee** commands for simple copy/pasting into a term.

    echo """
    [Unit]
    Description=Service that runs a GNUnet for the user gnunet
    After=network.target
    
    [Service]
    User=gnunet
    Type=simple
    ExecStart=/usr/local/lib/gnunet/libexec/gnunet-service-arm -c /etc/gnunet.conf
    
    [Install]
    WantedBy = multi.user.target
    """ | sudo tee /usr/local/share/gnunet/services/systemd/gnunet.service

**copy file into systemd dir:**

    sudo cp /usr/local/share/gnunet/services/systemd/gnunet.service /usr/lib/systemd/user/

**then 'enabled' with:**

    sudo systemctl enable /usr/lib/systemd/user/gnunet.service

...but wait, theres of course more:
its pointing to a place that no config file exists, so we will have to put one there:

    echo """[arm]
    SYSTEM_ONLY = NO
    USER_ONLY = NO
    
    [datastore-mysql]
    CONFIG = ~/.my.cnf
    
    [PATHS]
    DEFAULTCONFIG = ~/.config/gnunet.conf
    
    [transport]
    PLUGINS = tcp udp
    DISABLEV6 = YES
    
    [transport-tcp]
    PORT = 20202
    ADVERTISED_PORT = 20202
    
    [transport-udp]
    PORT = 20203
    
    [nat]
    ENABLE_UPNP = NO
    BEHIND_NAT = YES
    """ | sudo tee /etc/gnunet.conf

Note: you can change default ports above.  There's so many things one can tweak.  This isn't a tweaking guide, its a 'from zero to node you can use' guide.

**then you actually flip the 'on' switch with:**

    sudo systemctl daemon-reload; sudo systemctl start gnunet.service

...then you can see the action in..syslog. :/ 

    tail -f /var/log/syslog | grep -i gnunet

## Fixing suid bits
This solves the "vpn-9252 ERROR `gnunet-helper-vpn' is not SUID, refusing to run." etc seen in log.
**gnunet-suidfix** is installed but if I run as user, I get permissions errors. If I run with root, it tries to suid files in /usr/lib (gnunet bins aren't there). I never could make sense of this so I just copied out the files that it was trying to change and suid' them myself:

    sudo chmod u+s /usr/local/lib/gnunet/libexec/gnunet-helper-exit
    sudo chmod u+s /usr/local/lib/gnunet/libexec/gnunet-helper-nat-server
    sudo chmod u+s /usr/local/lib/gnunet/libexec/gnunet-helper-nat-client
    sudo chmod u+s /usr/local/lib/gnunet/libexec/gnunet-helper-vpn
    sudo chmod u+s /usr/local/lib/gnunet/libexec/gnunet-helper-dns
    sudo chmod u+s /usr/local/lib/gnunet/libexec/gnunet-service-dns

...Aaaand that's it!  If you made it this far, congrats slugger take a victory lap ;)

# Further caveats:
I had a pretty tight envrionment in which I was testing this and so this may not apply to you.  If you have a ton of messages in the log about IPV6 failing, read on:
i tried a mil things to get the log to shut up about ipv6 fails. there is a 'DISABLEV6' line in nearly every single conf file in /usr/local/share/gnunet/conf.d/. Which means in every subsection of the gnunet.conf...
and None had any effect whatsoever on the use of ipv6! LOL
**I finally disabled UDP entirely**
by removing 'udp' from '[transport] PLUGINS' and that stopped it from trying ipv6 for whatever reason.  
...So, edit your /etc/gnunet.conf file and look for the [transport] section. delete 'udp' from PLUGINS list.

## GNUNet-gtk
getting gnunet-gtk compiled and installed was as easy as this:

    cd ~/gnunet_installation
    git clone https://git.gnunet.org/gnunet-gtk.git/
    sudo apt -y install libgtk-3-dev libgladeui-dev libgl1-mesa-dev
    cd gnunet-gtk
    ./configure --prefix=$GNUNET_PREFIX --with-gnunet=$GNUNET_PREFIX
    make
    cd ~/gnunet_installation/gnunet-gtk
    sudo make install
    sudo ldconfig

## mysql datastore:
getting mysql datastore setup:
(note the 'identified by' part. you need to set a password.)

    sudo apt install mysql-server
    sudo mysql -u root -p
    #press enter at password prompt
    CREATE DATABASE gnunet;
    GRANT select,insert,update,delete,create,alter,drop,create \
    temporary tables ON gnunet.* TO gnunet@localhost identified by \
    'your_cool_password_here';
    FLUSH PRIVILEGES;

in ~, create file .my.conf. change user/pass to yours.

    echo """[client]
    user=$USER
    password=$the_password_you_like
    """ > ~/.my.cnf

NOTE: where this file is looked for is determined by **/etc/gnunet.conf**
If you followed by above instructions, there's a [datastore-mysql] section already there for your tweakage.
