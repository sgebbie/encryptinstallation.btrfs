#!/usr/bin/env bash


####################################################################################################
#	Run the Automated Grub refresh automatically after kernel updates.
#
# Reference:
#	https://help.ubuntu.com/community/ManualFullSystemEncryption/DetailedProcessSetUpBoot
#
#	*	Run with root permissions, i.e. with sudo.
#
#	*	Place in /usr/local/sbin/ and name it refreshgrub.
#
#	*	Automate in root's incrontab with the following line.
#
#		/boot/ IN_MODIFY,IN_NO_LOOP /usr/local/sbin/refreshgrub
#
#	*	Requires the following packages to be installed.
#			incron
#			libnotify-bin
#			yad
# 
####################################################################################################


#---------------------------------------------------------------------------------------------------
#	Initialise the script.
#
# Parameters
#	The parameters to the script.
#---------------------------------------------------------------------------------------------------


function initialise ()
{
	preventConcurrentRun "${@}"			# Prevent concurrent running.

	# Leave a message for every user.
	notifyUsers warning 'Grub update required' 'Grub update will be done automatically.\n\nDo not restart or shut down until you receive another message telling you that this has been done, even if the Software Updater asks you restart.\n\nDepending on your system, it could take several minutes.'

	waitForOthers					# Wait until we are free to proceed.

} # initialise


#---------------------------------------------------------------------------------------------------
#	Prevent concurrent runs.
#
# Lock this script before running. If already locked (i.e. already running), silently terminate.
#
# Parameters
#	The parameters to the script.
#---------------------------------------------------------------------------------------------------


function preventConcurrentRun ()
{
	# If PCR_CONCURRENT_FLAG is correctly set, it means that we are already locked and should proceed.
	# In that case, simply return from this function.
	# Otherwise, call the script recursively with a lock, indicating permission to proceed.
	if [[ "${PCR_CONCURRENT_FLAG}" != "${0}" ]]
	then
		# Call the script recursively, using itself as the exclusive lock.
		# Terminate silently if already locked; this can happen if the script is accidentally called twice.
		PCR_CONCURRENT_FLAG="${0}" flock --exclusive --conflict-exit-code=103 --nonblock -- "${0}" "${0}" "${@}"

		local -i RET=${?}			# Note the return code.
		(( RET == 103 )) && RET=0		# 103 means to fail silently, so reset to zero.
		exit ${RET}				# Exit the script with the correct return code.
	fi

} # preventConcurrentRun


#---------------------------------------------------------------------------------------------------
#	Send a message to all users currently logged in.
#
# Shown:
#	On the console in either &2 if error or &1 if not.
#	With notify-send.
#	yad, because notify-send doesn't always work.
#
# Parameters
#	1	Message type: "error", "info" or "warning".
#	2	The title of the message.
#	3	The message text.
#---------------------------------------------------------------------------------------------------


function notifyUsers ()
{
	local -r MESSAGE_TYPE=${1}
	local -r TITLE="${2}"
	local -r MESSAGE="${3}"

	local WHOLINE					# Results from the command "who".

	local WHOUSER					# The current user.
	local WHODISPLAY				# The user's display.

	# Find all users logged into the X terminal and notify them.
	w --short --no-header			|
	 tr --squeeze-repeats ' '		|
	  grep -E ' tty[0-9]+ '			|
	   grep -E ' :[0-9]+ '			|
	    while read WHOLINE
	    do
		# Extract the user and display.
		WHOUSER=$( cut --delimiter=' ' --field=1 <<<${WHOLINE} )
		WHODISPLAY=:$( cut --delimiter=':' --field=2 <<<${WHOLINE} | cut --delimiter=' ' --field=1 )

		# Put the message onto the console, in case we're running from there.
		if [[ ${MESSAGE_TYPE} == 'error' ]]
		then
			echo -e "refreshgrub: ${TITLE}\n${MESSAGE}" >&2
		else
			echo -e "refreshgrub: ${TITLE}\n${MESSAGE}"
		fi

		# Send the message to the user. notify-send doesn't work reliably, so we use yad as well.

		DISPLAY=${WHODISPLAY} sudo --user=${WHOUSER} notify-send --urgency=critical --icon=${MESSAGE_TYPE} "refreshgrub: ${TITLE}" "$( date +'%F %T' )\n\n${MESSAGE}" 2>/dev/null

		DISPLAY=${WHODISPLAY} sudo --user=${WHOUSER} yad --width=400 --image=dialog-${MESSAGE_TYPE} --window-icon=dialog-${MESSAGE_TYPE} --title="refreshgrub: ${TITLE}" --button=OK:0 --text="$( date +'%F %T' )\n\n${MESSAGE}" 2>/dev/null &

	    done

	wall "refreshgrub: ${MESSAGE}" 2>/dev/null	# Send the message to all console users.

	return 0					# Ignore previous errors.

} # notifyUsers


#---------------------------------------------------------------------------------------------------
#	Wait for the current installation process to finish, if not already done.
#---------------------------------------------------------------------------------------------------


function waitForOthers ()
{
	local -i NO_DPKG=0				# How long since termination?

	# Wait until any other installation has finished running for at least a short while.
	while (( NO_DPKG < 5 ))
	do
		sleep 1s
		if pgrep --count --newest --full 'dpkg|grub-mkconfig|update-initramfs|update-grub' >/dev/null
		then
			NO_DPKG=0			# Still running or restarted.
		else
			: $(( ++NO_DPKG ))		# Increment the counter.
		fi
	done

} # waitForOthers


#---------------------------------------------------------------------------------------------------
#	Refresh Grub.
#---------------------------------------------------------------------------------------------------


function refreshGrub ()
{
	reapplyGrubUpdates				# Update Grub, initramfs, etc.
	local -ir RET=${?}				# Note the return value.

	# Check for errors and leave a message.
	if (( RET ))
	then
		notifyUsers error 'Grub update failed' 'Grub update failed.\n\nPlease do not restart or shut down until you have manually run the following command, even if the Software Updater asks you restart.\n\nsudo refreshgrub'

		exit ${RET}				# Return with error.
	fi

	notifyUsers info 'Grub update succeeded' 'The Grub update has finished.\n\nYou are advised to restart the machine.'

} # refreshGrub


#---------------------------------------------------------------------------------------------------
#	Reapply Grub updates.
#---------------------------------------------------------------------------------------------------


function reapplyGrubUpdates ()
{
	local -i RET					# To hold return codes.

	# Copy boot modules to EFI
	mkdir --parents /boot/efi/EFI/ubuntu/
	RET=${?}
	(( RET )) && echo 'Failed to create boot modules folder in EFI.' >&2 && return ${RET}

	cp --recursive /boot/grub/x86_64-efi /boot/efi/EFI/ubuntu/
	RET=${?}
	(( RET )) && echo 'Failed to copy boot modules to EFI.' >&2 && return ${RET}

	# Install and repair Grub

	grub-install --target=x86_64-efi --uefi-secure-boot --efi-directory=/boot/efi --bootloader=ubuntu --boot-directory=/boot/efi/EFI/ubuntu --recheck /dev/[DRIVE]
	RET=${?}
	(( RET )) && echo 'Failed to reinstall Grub.' >&2 && return ${RET}

	grub-mkconfig --output=/boot/efi/EFI/ubuntu/grub/grub.cfg
	RET=${?}
	(( RET )) && echo 'Failed to reconfigure Grub.' >&2 && return ${RET}

	# Allow Ubuntu to boot
	cd /boot/efi/EFI
	RET=${?}
	(( RET )) && echo 'Failed to enter /boot/efi/EFI.' >&2 && return ${RET}

	[[ -d Boot ]] && rm --force --recursive Boot-backup && mv Boot Boot-backup
	RET=${?}
	# Ignore error code 1.
	(( RET > 1 )) && echo 'Failed to enter /boot/efi/EFI.' >&2 && return ${RET}

	# Prepare initramfs
	update-initramfs -ck all
	RET=${?}
	(( RET )) && echo 'Failed to prepare initrafms.' >&2 && return ${RET}

	return 0					# Because of "(( ... ))".

} # reapplyGrubUpdates


####################################################################################################
#	MAIN SCRIPT CONTROL
####################################################################################################


initialise						# Set up the script.
refreshGrub						# Do the work and let the user know.
