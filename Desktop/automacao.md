#!/bin/bash
# encoding: utf-8
 
##################################################
#		    Variaveis 			 #
##################################################
# Nome do Computador
HOSTN=Arch-VM
 
# Localização. Verifique o diretório /usr/share/zoneinfo/<Zone>/<SubZone>
LOCALE=America/Sao_Paulo
 
# Senha de Root do sistema após a instalação
ROOT_PASSWD=123456
 
########## Variáveis Para Particionamento do Disco
# ATENÇÃO, este script apaga TODO o conteúdo do disco especificado em $HD.
HD=/dev/sda
# Tamanho da Partição Boot: /boot
BOOT_SIZE=200
# Tamanho da Partição Root: /
ROOT_SIZE=10000
# Tamanho da Partição Swap:
SWAP_SIZE=2000
# A partição /home irá ocupar o restante do espaço livre em disco
 
# File System das partições
BOOT_FS=ext4
HOME_FS=ext4
ROOT_FS=ext4
 
# Pacote extras (não são obrigatórios)
EXTRA_PKGS='vim'
 
######## Variáveis menos suscetíveis a mudanças
KEYBOARD_LAYOUT=br-abnt2
LANGUAGE=pt_BR
 
######## Variáveis auxiliares. NÃO DEVEM SER ALTERADAS
BOOT_START=1
BOOT_END=$(($BOOT_START+$BOOT_SIZE))
 
SWAP_START=$BOOT_END
SWAP_END=$(($SWAP_START+$SWAP_SIZE))
 
ROOT_START=$SWAP_END
ROOT_END=$(($ROOT_START+$ROOT_SIZE))
 
HOME_START=$ROOT_END
 
##################################################
#		    Script 			 #
##################################################
# Carrega layout do teclado ABNT2
loadkeys $KEYBOARD_LAYOUT
 
#### Particionamento
echo "Inicializando o HD"
# Configura o tipo da tabela de partições (Ignorando erros)
parted -s $HD mklabel msdos &> /dev/null
 
# Remove qualquer partição antiga
parted -s $HD rm 1 &> /dev/null
parted -s $HD rm 2 &> /dev/null
parted -s $HD rm 3 &> /dev/null
parted -s $HD rm 4 &> /dev/null
 
# Cria partição boot
echo "Criando partição boot"
parted -s $HD mkpart primary $BOOT_FS $BOOT_START $BOOT_END 1>/dev/null
parted -s $HD set 1 boot on 1>/dev/null || ERR=1
 
# Cria partição swap
echo "Criando partição swap"
parted -s $HD mkpart primary linux-swap $SWAP_START $SWAP_END 1>/dev/null
 
# Cria partição root
echo "Criando partição root"
parted -s $HD mkpart primary $ROOT_FS $ROOT_START $ROOT_END 1>/dev/null
 
# Cria partição home
echo "Criando partição home"
parted -s -- $HD mkpart primary $HOME_FS $HOME_START -0 1>/dev/null
 
#### Preparando Partições
# Formata partições root, home e boot para o File System especificado
echo "Formatando partição boot"
mkfs.$BOOT_FS /dev/sda1 -L Boot 1>/dev/null
echo "Formatando partição root"
mkfs.$ROOT_FS /dev/sda3 -L Root 1>/dev/null
echo "Formatando partição home"
mkfs.$HOME_FS /dev/sda4 -L Home 1>/dev/null
# Cria e inicia a swap
echo "Formatando partição swap"
mkswap /dev/sda2
swapon /dev/sda2
 
echo "Montando partições"
# Monta partição root
mount /dev/sda3 /mnt
# Monta partição boot
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
# Monta partição home
mkdir /mnt/home
mount /dev/sda4 /mnt/home
 
#### Instalação
echo "Configurando pacman"
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bkp
# Comenta todos os servers
sed "s/^Ser/#Ser/" /etc/pacman.d/mirrorlist > /tmp/mirrors
# Descomenta apenas servers do Brasil
sed '/Brazil/{n;s/^#//}' /tmp/mirrors > /etc/pacman.d/mirrorlist
 
# Multilib apenas para 64 bits
if [ "$(uname -m)" = "x86_64" ]
then
	cp /etc/pacman.conf /etc/pacman.conf.bkp
	# Adiciona o Multilib 
	sed '/^#\[multilib\]/{s/^#//;n;s/^#//;n;s/^#//}' /etc/pacman.conf > /tmp/pacman
	mv /tmp/pacman /etc/pacman.conf
 
fi
 
echo "Rodando pactrap base base-devel"
pacstrap /mnt base base-devel
echo "Rodando pactrap grub-bios e pacotes extras"
pacstrap /mnt grub-bios $EXTRA_PKGS
echo "Rodando genfstab"
genfstab -p /mnt >> /mnt/etc/fstab
 
#### Entra no novo sistema (chroot)
arch-chroot /mnt << EOF
# Configura hostname
echo $HOSTN > /etc/hostname
cp /etc/hosts /etc/hosts.bkp
sed 's/localhost$/localhost '$HOSTN'/' /etc/hosts > /tmp/hosts
mv /tmp/hosts /etc/hosts
 
# Configura layout do teclado
echo 'KEYMAP='$KEYBOARD_LAYOUT > /etc/vconsole.conf
echo 'FONT=lat0-16' >> /etc/vconsole.conf
echo 'FONT_MAP=' >> /etc/vconsole.conf
 
# Configura locale.gen
cp /etc/locale.gen /etc/locale.gen.bkp
sed 's/^#'$LANGUAGE'/'$LANGUAGE/ /etc/locale.gen > /tmp/locale
mv /tmp/locale /etc/locale.gen
locale-gen
 
# Configura locale.conf
export LANG=$LANGUAGE'.utf-8'
echo 'LANG='$LANGUAGE'.utf-8' > /etc/locale.conf
echo 'LC_COLLATE=C' >> /etc/locale.conf
echo 'LC_TIME='$LANGUAGE'.utf-8' >> /etc/locale.conf
 
# Configura hora
ln -s /usr/share/zoneinfo/$LOCALE /etc/localtime
echo $LOCALE > /etc/timezone
hwclock --systohc --utc
 
# Configura rede (DHCP via eth0)
cp /etc/rc.conf /etc/rc.conf.bkp
sed 's/^# interface=/interface=eth0/' /etc/rc.conf > /tmp/rc.conf
mv /tmp/rc.conf /etc/rc.conf
 
# Configura ambiente ramdisk inicial
mkinitcpio -p linux
 
# Instala e gera configuração do GRUB Legacy
grub-install /dev/sda
grub-mkconfig -o /boot/grub/grub.cfg
 
# Altera a senha do usuário root
echo -e $ROOT_PASSWD"\n"$ROOT_PASSWD | passwd
EOF
 
echo "Umounting partitions"
umount /mnt/{boot,home,}
reboot
