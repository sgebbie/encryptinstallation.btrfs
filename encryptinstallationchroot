#!/usr/bin/env bash


####################################################################################################
#
#	ENCRYPT INSTALLATION
#
# This script is to be run only by encryptinstallation, which in turn is run as a helper for
# https://help.ubuntu.com/community/ManualFullSystemEncryption
#
# Never run this manually or in any other way, as it will damage your system.
#
# Parameters
#	1	Hibernation enabled (true or false)
#
# Return code
#	0	Successful process.
#	Anything else means an error.
#
####################################################################################################


#---------------------------------------------------------------------------------------------------
#	Set up the script.
#---------------------------------------------------------------------------------------------------


function initialise ()
{
	trap ctrlC SIGINT				# Trap Ctrl+C.

	# Miscellaneous items for display.
	declare -gr SPACER='----------------------------------------------------------------------------------------------------'
	declare -gr E=$'\e[1;31;103m'			# E for Error: highlighted text.
	declare -gr R=$'\e[0m'				# R for Reset.

	declare -gr HIBERNATION_CHOSEN=${1}		# true or false.

	local -i RET					# To catch errors.

	mount --types=proc proc /proc			# Mount /proc.
	RET=${?}
	(( RET )) && error 'chroot: Unable to mount /proc.' ${RET}

	mount --types=sysfs sys /sys			# Mount /sys.
	RET=${?}
	(( RET )) && error 'chroot: Unable to mount /sys.' ${RET}

} # initialise


#---------------------------------------------------------------------------------------------------
#	Trap for Ctrl+C.
#---------------------------------------------------------------------------------------------------


function ctrlC ()
{
	cat <<-END
		${SPACER}

		        ********        Terminated with Ctrl+C.        ********

		${SPACER}
	END

	exit 2

} # ctrlC


#---------------------------------------------------------------------------------------------------
#	Display an error messsage.
#
# Parameter
#	1	The error message.
#	2	The exit code (numeric); if absent, won't exit.
#---------------------------------------------------------------------------------------------------


function error ()
{
	local -r ERROR="${1}"
	local -r EXIT_CODE=${2}

	echo -e "\n${E}${ERROR}${R}\n" >&2		# Display the message, highlighted.

	[[ -n ${EXIT_CODE} ]] && exit ${EXIT_CODE}	# Exit with the code if given.

} # error


#---------------------------------------------------------------------------------------------------
#	Complete the installation.
#---------------------------------------------------------------------------------------------------


function completeInstallation ()
{
	installApplications				# Install the required applications.
	setAutoGrubUpdate				# Set Grub to auto-update.
	return ${?}

} # completeInstallation


#---------------------------------------------------------------------------------------------------
#	Install the required applications.
#---------------------------------------------------------------------------------------------------


function installApplications ()
{
	local -i RET					# To catch errors.

	apt update --quiet >/dev/null			# Update the repositories.
	RET=${?}
	(( RET )) && error 'Failed to update the repositories.' ${RET}

	# Install the required applications.
	apt install --assume-yes --quiet incron libnotify-bin yad
	RET=${?}
	(( RET )) && error 'Failed to install required applications for chroot.' ${RET}

	echo root >/etc/incron.allow			# Allow root to use incron.
	RET=${?}
	(( RET )) && error 'Unable to enable incron for auto-update for Grub.' ${RET}

} # installApplications


#---------------------------------------------------------------------------------------------------
#	Set Grub to auto-update.
#---------------------------------------------------------------------------------------------------


function setAutoGrubUpdate ()
{
	local -i RET					# To catch errors.

	# Start refreshgrub automatically when /boot has been changed.
	echo '/boot/ IN_MODIFY,IN_NO_LOOP /usr/local/sbin/refreshgrub' | incrontab -
	RET=${?}
	(( RET )) && error 'Failed to automate Grub update.' ${RET}

	# Let the user know what's happening.
	cat <<-END
		${SPACER}

			Grub refresh is starting. It might take a couple of minutes.

		${SPACER}

	END

	refreshgrub					# Refresh Grub now.
	RET=${?}
	if (( RET ))
	then
		cat <<-END

			${SPACER}

			${E}There were errors with Grub refresh.${R}

			Please check the previous output.
			You may safely ignore any error about "wall".

			If you feel that it's safe to proceed, go ahead.

			If not, seek help, because the installation process has finished, but
			your computer is not ready to be rebooted yet.

		END
		read -rp 'Press Enter to continue. '

		return ${RET}
	fi

	# Let the user know what's happening.
	cat <<-END

		${SPACER}

		        Grub refresh has finished.
	END

} # setAutoGrubUpdate


####################################################################################################
#	MAIN SCRIPT CONTROL
####################################################################################################


initialise "${@}"					# Continue setting up.
completeInstallation					# Complete the installation.
exit ${?}						# Send back the return code.
