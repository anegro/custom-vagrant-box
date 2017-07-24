# Custom Vagrant Box

A continuación se resumen de todos los pasos a seguir para la preparación de una *Vagrant Box* personalizada. Como ejemplo, se crea una caja para Ubuntu.

**Motivación**: Las útlimas cajas publicadas por Canonical no respetan el estilo tradicional de Vagrant, entre otras cosas, al no incluir al usuario **vagrant** con clave **vagrant** ni la clave criptográfica insegura para el acceso por SSH.

## Requisitos

- VirtualBox
- Vagrant

## Crear la máquina virtual y configurar el hardware

Crear una nueva máquina virtual con los siguientes ajustes:

* Nombre: vagrant-ubuntu64
* Tipo: Linux
* Versión: Ubuntu64
* Tamaño de memoria: 512MB (por ejemplo)
* Nuevo disco virtual: [Tipo: VMDK, Tamaño: 40 GB]

Modificar los ajustes del hardware para mejorar el rendimiento y para permitir el acceso por SSH:

* Deshabilitar audio
* DeshabilitarUSB
* Ajustar el primer adaptador de red en modo NAT
* Añadir la siguiente regla de reenvío de puertos: [Nombre: SSH, Protocolo: TCP, IP Anfitrión: en blanco, Puerto Anfitrión: 2222, IP Huésped: en blanco, Puerto Huésped: 22]

Cargar la ISO y arrancar.

## Instalar el sistema operativo

Al final de la instalación, crear el usuario `vagrant` con clave `vagrant`.

## Configurar la clave de root

Vagrant no la necesita, pero es recomendable utilizar alguna clave conocida, por ejemplo "vagrant".

~~~
sudo passwd root
~~~

## Añadir vagrant a la lista de sudoers

sudo visudo -f /etc/sudoers.d/vagrant

~~~.sh
# add vagrant user
vagrant ALL=(ALL) NOPASSWD:ALL
~~~

## Actualizar el sistema y reiniciar

~~~
sudo apt-get update -y
sudo apt-get upgrade -y
sudo reboot
~~~

## Instalar la clave de vagrant

~~~
mkdir -p /home/vagrant/.ssh
chmod 0700 /home/vagrant/.ssh
wget --no-check-certificate \
    https://raw.github.com/mitchellh/vagrant/master/keys/vagrant.pub \
    -O /home/vagrant/.ssh/authorized_keys
chmod 0600 /home/vagrant/.ssh/authorized_keys
chown -R vagrant /home/vagrant/.ssh
~~~

## Instalar y configurar el servidor SSH

Instalar el servicio:

~~~
sudo apt-get install -y openssh-server
~~~

Editar la configuración:

~~~
sudo nano /etc/ssh/sshd_config
~~~

Comprobar que están configuradas las líneas:

~~~
AuthorizedKeysFile %h/.ssh/authorized_keys
UseDNS no
~~~

Reiniciar servicio:

~~~
sudo service ssh restart
~~~

## Instalar las *Guest Additions*

Instalar kernel y cabeceras para máquinas virtuales:

~~~
sudo apt-get install -y linux-image-virtual linux-headers-virtual
~~~

Instalar herramientas de compilación:

~~~
sudo apt-get install -y gcc build-essential
~~~

Insertar el CD con las *Guest Additions* e instalarlas:

~~~
sudo mount /dev/cdrom /mnt 
cd /mnt
sudo ./VBoxLinuxAdditions.run
~~~

## Revisar la configuración de red

Editar `/etc/default/grub` y añadir:

~~~
GRUB_CMDLINE_LINUX_DEFAULT="net.ifnames=0 biosdevname=0"
~~~

Seguidamente, ejecutar `sudo update-grub`.

Revisar el archivo `/etc/network/interfaces`:

~~~.sh
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

# The loopback network interface
auto lo
iface lo inet loopback

# Source interfaces
# Please check /etc/network/interfaces.d before changing this file
# as interfaces may have been defined in /etc/network/interfaces.d
# NOTE: the primary ethernet device is defined in
# /etc/network/interfaces.d/eth0
# See LP: #1262951
source /etc/network/interfaces.d/*.cfg

#VAGRANT-BEGIN
iface eth1 inet manual
iface eth2 inet manual
iface eth3 inet manual
#VAGRANT-END
~~~

Revisar el archivo `/etc/network/interfaces.d/eth0.cfg`:

~~~.sh
# The primary network interface
auto eth0
iface eth0 inet dhcp
~~~

## Empaquetar la caja

En primer lugar, rellenar con ceros el espacio libre:

~~~
sudo dd if=/dev/zero of=/EMPTY bs=1M
sudo rm -f /EMPTY
~~~

Apagar la máquina y crear la caja con el comando:

~~~
vagrant package --base vagrant-ubuntu64
~~~

Cuando el proceso termine, se habrá creado un archivo llamado `package.box`.

Opcionalmente, podemos indicar un Vagranfile personalizado:

~~~
vagrant package --base vagrant-ubuntu64 --vagrantfile myVagrantfile
~~~

Para máquinas de escritorio, un Vagranfile adecuado puede ser el siguiente:

~~~.rb
# The contents below (if any) are custom contents provided by the
# Packer template during image build.
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
    config.vm.define "vagrant-xenial-desktop-es"
    config.vm.box = "xenial-desktop-es"

    config.vm.provider :virtualbox do |v, override|
        v.gui = true
        v.customize ["modifyvm", :id, "--memory", 1024]
        v.customize ["modifyvm", :id, "--cpus", 1]
        v.customize ["modifyvm", :id, "--vram", "256"]
        v.customize ["setextradata", "global", "GUI/MaxGuestResolution", "any"]
        v.customize ["setextradata", :id, "CustomVideoMode1", "1024x768x32"]
        v.customize ["modifyvm", :id, "--ioapic", "on"]
        v.customize ["modifyvm", :id, "--rtcuseutc", "on"]
        v.customize ["modifyvm", :id, "--accelerate3d", "on"]
        v.customize ["modifyvm", :id, "--clipboard", "bidirectional"]
    end

    ["vmware_fusion", "vmware_workstation"].each do |provider|
      config.vm.provider provider do |v, override|
        v.gui = true
        v.vmx["memsize"] = "1024"
        v.vmx["numvcpus"] = "1"
        v.vmx["cpuid.coresPerSocket"] = "1"
        v.vmx["ethernet0.virtualDev"] = "vmxnet3"
        v.vmx["RemoteDisplay.vnc.enabled"] = "false"
        v.vmx["RemoteDisplay.vnc.port"] = "5900"
        v.vmx["scsi0.virtualDev"] = "lsilogic"
        v.vmx["mks.enable3d"] = "TRUE"
      end
    end
end
~~~


## Referencias

- [Building a Vagrant Box from Start to Finish](https://blog.engineyard.com/2014/building-a-vagrant-box)

