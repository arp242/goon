#!/usr/bin/env zsh
[ "${ZSH_VERSION:-}" = "" ] && echo >&2 "Only works with zsh" && exit 1
setopt err_exit no_unset pipefail

ssh_port=9927

start-machine() {
	local toplevel=${1:-0}
	local arch=x86_64
	local version=''
	local img=$root/disks/9legacy.qcow2
	local pid=$root/run/$machine-$version.pid
	local ssh_ip=$(resolve-host $ssh_host)
	local procname=goon-$machine

	has-qemu $arch || return 1

	mkdir -p $root/run
	if has-pid $pid $procname; then
		print "$progname: $machine already started; nothing to do"
		return 0
	fi

	# No display unless this script is called directly.
	local display=(-display 'none')
	(( $toplevel )) && display=(
		-display  'gtk,grab-on-hover=off,zoom-to-fit=off'
		-device   'VGA,vgamem_mb=64'
		# Prevents the mouse from being grabbed when clicking in the guest; QEMU
		# will be able to report the mouse position without having to grab it.
		# Also overrides PS/2 mouse emulation when activated.
		-device   'usb-tablet,bus=input.0'
	)

	local iso=disks/9legacy.iso
	print "$progname: starting QEMU VM from $img:t"
	local cmd=(qemu-system-$arch
		-enable-kvm
		-cpu   'host,kvm=on'
		-hda   $img
		-vga   std
		-m     1024
		-net   nic,model=virtio,macaddr=00:20:91:37:33:77
		-net user

		#-cdrom $iso -boot  d

		# -enable-kvm
		# -machine    'pc,smm=off,vmport=off'
		# -rtc        'base=localtime,clock=host,driftfix=slew'
		# -cpu        'host,kvm=on'
		# -smp        'cores=1,threads=2,sockets=1'
		# -m          '2G'
		# -name       "$machine,process=$procname"
		# -pidfile    "$pid"
		#-device     'usb-ehci,id=input'

		# $display

		# Hard drive
		# -device     "virtio-blk-pci,drive=SystemDisk"
		# -drive      "id=SystemDisk,if=none,file=$img,format=qcow2"

		# Network
		# -device e1000,netdev=mynet0
		# -netdev user,id=mynet0,hostfwd=tcp:$ssh_ip:$ssh_port-:22
	)
	${cmd[@]} &
}

if [[ $zsh_eval_context = 'toplevel' ]]; then
	source $0:P:h:h/goon
	read-machine $0:P:t
	start-machine 1
fi
