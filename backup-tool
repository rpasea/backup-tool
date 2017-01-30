#!/bin/bash
shopt -s expand_aliases

CONF=/home/osmc/backup/conf/backup.conf

REPORT="report_$(date -I)"

pushd `dirname $0` > /dev/null
SCRIPTPATH=`pwd -P`
popd > /dev/null

CMD="$SCRIPTPATH/$(basename $0) backup"
PIDFILE="$SCRIPTPATH/pidfile"

alias errcho='>&2 echo'

LOCKFILE="/var/lock/`basename $0`"
LOCKFD=99

# PRIVATE
_lock()             { flock -$1 $LOCKFD; }
_no_more_locking()  { _lock u; _lock xn && rm -f $LOCKFILE; }
_prepare_locking()  { eval "exec $LOCKFD>\"$LOCKFILE\""; trap _no_more_locking EXIT; }

# ON START
_prepare_locking

# PUBLIC
exlock_now()        { _lock xn; }  # obtain an exclusive lock immediately or fail
exlock()            { _lock x; }   # obtain an exclusive lock
shlock()            { _lock s; }   # obtain a shared lock
unlock()            { _lock u; }   # drop a lock


function check_deps {
	ok=true
	missing=""

	type crontab &> /dev/null
	if [ $? -ne 0 ]; then
		ok=false
		missing=$missing" cron"
	fi
	
	type crudini &> /dev/null
	if [ $? -ne 0 ]; then
		ok=false
		missing=$missing" crudini"
	fi
	
	type rsync &> /dev/null
	if [ $? -ne 0 ]; then
		ok=false
		missing=$missing" rsync"
	fi
	
	type mail &> /dev/null
	if [ $? -ne 0 ]; then
		ok=false
		missing=$missing" mailutils"
	fi
	
	type uuencode &> /dev/null
	if [ $? -ne 0 ]; then
		ok=false
		missing=$missing" sharutils"
	fi
	
	type cifsdd &> /dev/null
	if [ $? -ne 0 ]; then
		ok=false
		missing=$missing" cifs-utils"
	fi

	if [ $ok = false ]; then
		errcho "Missing packages:"$missing
		return 1
	else
		errcho "All packages are installed"
		return 0
	fi
}

function install_deps {
	if [[ $EUID -ne 0 ]]; then
		errcho "This function must be run as root" 
		return 1
	fi

	apt-get -y install cron crudini rsync mailutils sharutils cifs-utils
}

function get_cron_from_conf {
	crudini --get $CONF cron &> /dev/null
	
	if [ $? -ne 0 ]; then
		errcho "cron section not found in config file. Check that "$CONF" exists and it contains the 'cron' section" 
		return 1
	fi
	
	daily_time=$(crudini --get $CONF cron daily_time 2> /dev/null)
	schedule=$(crudini --get $CONF cron schedule 2> /dev/null)
	
	if [ ! -z "$schedule" ] ; then
		errcho "Cron schedule parameter set to: $schedule"
		
		if [ ! -z "$daily_time" ] ; then
			errcho "Cron daily_time parameter set to: "$daily_time
			errcho "Both 'schedule' and 'daily_time' parameters are set. The 'schedule' parameter has precedence'"
		fi
		
		echo "$schedule"
	elif [ ! -z "$daily_time" ] ; then
		errcho "Cron daily_time parameter set to: "$daily_time
		hour=$(echo $daily_time | cut -d ":" -s --output-delimiter=" " -f1)
		minute=$(echo $daily_time | cut -d ":" -s --output-delimiter=" " -f2)
		
		if ! (( 0 <= $hour && $hour <= 23 )) 2> /dev/null ; then
			errcho "daily_time parameter must be in hh:mm format"
			return 1
		fi
		
		if ! (( 0 <= $minute && $minute <= 59 )) 2> /dev/null ; then
			errcho "daily_time parameter must be in hh:mm format"
			return 1
		fi
		
		echo "$minute $hour * * *"
	else
		errcho "Neither cron schedule nor daily_time set in config file. Please set one of them"
	fi
}

function set_cron {
	schedule="$(get_cron_from_conf)"
	
	if [ $? -ne 0 ] ; then
		return 1
	fi
	
	errcho "Setting cron schedule $schedule"
	
	crontab -l 2> /dev/null | grep -v "$CMD" > mycron
	
	cron_line="$schedule $CMD"
	echo "$cron_line" >> mycron
	
	crontab mycron
	rm mycron
}

function backup_source {
	source="$1"
	target="$2"
	
	echo "Performing backup of '$source' to '$target'" 2>&1 | tee -a $REPORT
	rsync -av "$source" "$target" 2>&1 | tee -a $REPORT
	ret_code=$?
	echo "*******************************************" 2>&1 | tee -a $REPORT
	return $ret_code
}

function send_mail {
	status=$1
	
	recipients=$(crudini --get $CONF backup recipients)
	if [ -z "$recipients" ] ; then
		errcho "No recipients specified in config file. Not sending mail."
		return 1
	fi
	
	subject="[backup_tool]$1"
	uuencode "$REPORT" "$REPORT.ext" | mail -s $subject $recipients
	if [ $? -ne 0 ]; then
		errcho "Could not send mail to: $recipients" 
		return 1
	else
		errcho "Successfully sent mail to: $recipients"
	fi
}

function perform_backup {
	echo > "$REPORT"

	crudini --get $CONF backup &> /dev/null
	if [ $? -ne 0 ]; then
		errcho "'backup' section not found in config file. Check that "$CONF" exists and it contains the 'backup' section" 
		return 1
	fi
	
	sources=$(crudini --get $CONF backup sources 2> /dev/null)
	if [ -z "$sources" ] ; then
		errcho "No sources specified in config file. Add source folders as a comma separated list in the 'sources' parameter in the 'backup' section." 2>&1 | tee -a $REPORT
		return 1
	fi
	
	target=$(crudini --get $CONF backup target 2> /dev/null)
	if [ -z "$target" ] ; then
		errcho "No target specified in config file. Add a target folder in the 'target' parameter in the 'backup' section." 2>&1 | tee -a $REPORT
		return 1
	fi
	
	if [ ! -d "$DIRECTORY" ]; then
		mkdir -p "$target"
		if [ $? -ne 0 ]; then
			errcho "Failed creating target folder $target. Try to create it manually an run again." 2>&1 | tee -a $REPORT
			return 1
		fi
	fi
	
	IFS=',' read -ra SOURCE <<< "$sources"
	rc=0
	for source in "${SOURCE[@]}"; do
		backup_source "$source" "$target"
		let "rc|=$?"
	done
	
	return $rc
}

function permanent_mount {
	if [[ $EUID -ne 0 ]]; then
		errcho "This function must be run as root" 
		return 1
	fi
	
	echo "Current mount table:"
	cat /etc/fstab
	printf "\n"
	echo "Available devices:"
	
	output=$(blkid)
	IFS=$'\n' read -d '' -ra DEVICES <<< "$output"
	for i in "${!DEVICES[@]}"; do
		dev="${DEVICES[$i]}"
		dev_id=$(echo $dev | cut -d':' -f1)
		size=$(blockdev --getsize64 $dev_id)
		let "size/=1024"
		let "size/=1024"
		OPTIONS[$i]="$i) $dev_id size: $size MB"
		echo ${OPTIONS[$i]}
	done
	while true; do
		echo "Select option:"
		read device_option
		if [ "$device_option" -eq "$device_option" ] 2>/dev/null ; then
			if (( 0 <= $device_option && $device_option < ${#OPTIONS[@]} )) ; then
				break
			fi
		fi
		echo "Invalid option. Please select number from range 0:$((${#OPTIONS[@]}-1))"
	done
	selected_id=$(echo ${DEVICES[$device_option]} | cut -d':' -f1)
	echo "Selected device $selected_id. Enter a path to be used as mount point:" 
	read mount_point
	if mount | grep $mount_point > /dev/null; then
		errcho "Path already mounted, choose another one"
		return 1
	fi
	if [ -f "$mount_point" ] ; then
		errcho "Specified mount point exists as file. It must be an empty folder"
		return 1
	fi
	if [ -d "$mount_point" ]; then
		if [ "$(ls -A $mount_point)" ]; then
			errcho "Path not empty. Use an empty folder"
			return 1
		fi
	else
		mkdir -p $mount_point
		if [ $? -ne 0 ]; then
			errcho "Cannot create mount_point. Try to create it manually."
			return 1
		fi
		chmod a+r $mount_point
	fi
	
	read_only=0
	while true; do
		echo "Read only mount? (y/n)"
		read option
		case $option in
			y)
				read_only=1
				break
				;;
			n)
				read_only=0
				break
				;;
			*)
				echo "Invalid option. Please select number from y/n"
				;;
		esac
	done
	
	cat /etc/fstab > fstab.bkp
	
	values=$(blkid $selected_id -o export)
	for val in $values; do
		export $val
	done
	
	if (( $read_only )); then
		fstab_line="UUID=$UUID	$mount_point	$TYPE	ro,exec,nofail,auto,async	0	0"
	else
		fstab_line="UUID=$UUID	$mount_point	$TYPE	rw,exec,nofail,auto,async	0	0"
	fi
	echo $fstab_line >> /etc/fstab
	
	mount -a
	if [ $? -ne 0 ]; then
		errcho "Cannot mount device. Reverting"
		mv fstab.bkp /etc/fstab
		mount -a
		return 1
	fi
}

function permanent_mount_samba {
	if [[ $EUID -ne 0 ]]; then
		errcho "This function must be run as root" 
		return 1
	fi
	
	echo "Current mount table:"
	cat /etc/fstab
	printf "\n"
	
	echo "Provide network URL for samba server:"
	read url
	if ! [[ $url == //* ]]; then
		url="//$url"
	fi
	
	credentials=""
	echo "Provide samba share username:"
	read username
	if [ ! -z "$username" ] ; then
		credentials="$credentials,username=$username"
	fi
	
	echo "Provide samba share password:"
	read password
	credentials="$credentials,password=$password"
	
	echo "Will try to mount path $url with credentials:($credentials). Enter a local path to be used as mount point:" 
	read mount_point
	if mount | grep $mount_point > /dev/null; then
		errcho "Path already mounted, choose another one"
		return 1
	fi
	if [ -f "$mount_point" ] ; then
		errcho "Specified mount point exists as file. It must be an empty folder"
		return 1
	fi
	if [ -d "$mount_point" ]; then
		if [ "$(ls -A $mount_point)" ]; then
			errcho "Path not empty. Use an empty folder"
			return 1
		fi
	else
		mkdir -p $mount_point
		if [ $? -ne 0 ]; then
			errcho "Cannot create mount_point. Try to create it manually."
			return 1
		fi
		chmod a+r $mount_point
	fi
	
	read_only=0
	while true; do
		echo "Read only mount? (y/n)"
		read option
		case $option in
			y)
				read_only=1
				break
				;;
			n)
				read_only=0
				break
				;;
			*)
				echo "Invalid option. Please select number from y/n"
				;;
		esac
	done
	
	cat /etc/fstab > fstab.bkp
	
	if (( $read_only )); then
		fstab_line="$url	$mount_point	cifs	ro,exec,nofail,auto,async$credentials	0	0"
	else
		fstab_line="$url	$mount_point	cifs	rw,exec,nofail,auto,async$credentials	0	0"
	fi
	echo $fstab_line >> /etc/fstab
		
	mount -a
	if [ $? -ne 0 ]; then
		errcho "Cannot mount device. Reverting"
		mv fstab.bkp /etc/fstab
		mount -a
		return 1
	fi
}

function permanent_umount {
	if [[ $EUID -ne 0 ]]; then
		errcho "This function must be run as root" 
		return 1
	fi
	echo "Current mount table:"
	i=0
	while read -r line ; do echo "$i) $line" ; let "i+=1" ; done < /etc/fstab
	
	while true; do
		echo "Select option:"
		read device_option
		if [ "$device_option" -eq "$device_option" ] 2>/dev/null ; then
			if (( 0 <= $device_option && $device_option < $i )) ; then
				break
			fi
		fi
		echo "Invalid option. Please select number from range 0:$(($i-1))"
	done
	
	cat /etc/fstab > fstab.bkp
	
	line=$(sed -n "$(($device_option+1))p" /etc/fstab)
	arr=($line)
	mount=${arr[1]}
	umount $mount
	if [ $? -ne 0 ]; then
		errcho "Cannot umount device."
		return 1
	fi
	
	
	sed --in-place "$(($device_option+1))d" /etc/fstab
	mount -a
	if [ $? -ne 0 ]; then
		errcho "Cannot umount device. Reverting"
		mv fstab.bkp /etc/fstab
		mount -a
		return 1
	fi
}

while [[ $# -gt 0 ]] ; do
	key="$1"

	case $key in
		schedule)
			set_cron
			exit $?
			;;
		backup)
			exlock_now
			if [ $? -ne 0 ]; then
				echo "Script already running"
				exit 1
			fi
			echo "$BASHPID" > $PIDFILE
			check_deps
			if [ $? -ne 0 ]; then
				rm $PIDFILE
				exit 1
			fi
			perform_backup
			rc=$?
			if [ $rc -ne 0 ]; then
				send_mail "FAILED"
			else
				send_mail "SUCCESS"
			fi
			rm $PIDFILE
			exit $rc
			;;
		"mount")
			permanent_mount
			exit $rc
			;;
		"mount-samba")
			permanent_mount_samba
			exit $rc
			;;
		"umount")
			permanent_umount
			exit $?
			;;
		*)
            errcho "Unknown command $key"
			exit 1
			;;
	esac
	shift # past argument or value
done
