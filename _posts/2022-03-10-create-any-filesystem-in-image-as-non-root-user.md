---
layout: post
title: "create any filesystem in disk image as a non-root user"
---

## why
Sometimes I have to create some virtual disk image and create filesystems in it(e.g:  
Windows answerfile image) and attach it to my test libvirt/KVM Guest, but when I do  
creating filesystem I must do losetup and mkfs with root privileges(sudo). and I don't  
always get root permissions,, sh\*t  

## how
by googling I got some ideas, for example mke2fs, but it only could be used to create  
ext* filesystem. so seems I have to figure it out myself..  

luckily I got one idea: split disk head and partition after creating partition done,  
and got two files(assume only created one partition): img-head, img-part; then do:  
```
mkfs.$fstype img-part
cat img-head img-part >disk.img
```

and the exciting thing is that it actually works, here's the full script:  
(now it only create one partition and satisfy me)  
```
dd_range() {
        local if=$1
        local of=$2
        local SKIP=${3:-0}
        local SIZE=$4
        local BS=$((8 * 1024))
        local Q= R= Q2= R2= NREAD=

        [[ -z "$of" ]] && return 1
        local fsize=$(stat -c%s "$if") || return $?

        ((fsize <= BS)) && BS=$SKIP
        Q=$((SKIP/BS))  #quotient
        R=$((SKIP%BS))  #residue
        NREAD=$((BS-R))
        dd if=$if ibs=$BS skip=$Q count=1 | tail -c $NREAD >$of

        if [[ -z "$SIZE" ]]; then
                dd if=$if bs=$BS skip=$((Q+1)) oflag=append conv=notrunc of=$of
        else
                if ((NREAD > SIZE)); then
                        truncate --size=${SIZE} $of
                        return 0
                fi
                let SIZE-=$NREAD
                Q2=$((SIZE/BS))  #quotient
                R2=$((SIZE%BS))  #residue
                ((Q2>0)) && dd if=$if bs=$BS skip=$((Q+1)) count=$Q2 oflag=append conv=notrunc of=$of
                ((R2>0)) && dd if=$if bs=$BS skip=$((Q+1+Q2)) count=1 | head -c $R2 >>$of
        fi
}

create_vdiskn() {
	local path=$1
	local dsize=$2
	local fstype=$3
	local imghead=img-head-$$
	local imgtail=img-tail-$$
	local fn=${FUNCNAME[0]}

	echo -e "\n[$fn:info] creating disk and partition"
	dd if=/dev/null of=$path bs=1${dsize//[0-9]/} seek=${dsize//[^0-9]/}
	printf "o\nn\np\n1\n\n\nw\n" | fdisk "$path"
	partprobe "$path"

	read pstart psize < <( LANG=C parted -s $path unit B print | sed 's/B//g' |
		awk -v P=1 '/^Number/{start=1;next}; start {if ($1==P) {print $2, $4}}' )
	echo -e "\n[$fn:info] split disk head and partition($pstart:$psize)"
	dd if=$path of=$imghead bs=${pstart} count=1
	dd_range $path $imgtail $((pstart)) ${psize}

	echo -e "\n[$fn:info] making fs($fstype)"
	mkfs.$fstype $MKFS_OPT "$imgtail"

	echo -e "\n[$fn:info] concat image-head and partition"
	cat $imghead $imgtail >$path
	rm -vf $imghead $imgtail
}

[[ $# -lt 3 ]] && {
	cat <<-COMM
	Usage: [MKFS_OPT=xxx] $0 <image> <size> <fstype>

	Examples:
	  $0 usb.img 256M vfat
	  $0 ext4.img 4G ext4
	  MKFS_OPT="-f -i attr=2,size=512" $0 xfs.img 4G xfs
	COMM
	exit 1
}
create_vdiskn "$@"
```

## see also
see also: [create-vdisk.sh](https://github.com/tcler/kiss-vm-ns/blob/master/utils/create-vdisk.sh)  
welcome improve it or report bug/feedback/..
