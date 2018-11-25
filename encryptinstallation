#!/usr/bin/env bash


####################################################################################################
#
#	ENCRYPT INSTALLATION
#
# This script is to be run only as instructed in Manual Full System Encryption for Ubuntu. See
# https://help.ubuntu.com/community/ManualFullSystemEncryption
#
# If this script is run anywhere or any time else, it will probably wreak havoc on your system,
# leaving it unbootable and deleting data.
#
# In other words, follow the instructions.
#
####################################################################################################


#===================================================================================================
#
#	Section: Initialise
#
#===================================================================================================


#---------------------------------------------------------------------------------------------------
#	Set up the script.
#---------------------------------------------------------------------------------------------------


function initialise ()
{
	trap ctrlC SIGINT				# Trap Ctrl+C.

	# Miscellaneous items for display.
	declare -gr SPACER='----------------------------------------------------------------------------------------------------'
	declare -gr E=$'\e[1;31;103m'			# E for Error: highlighted text.
	declare -gr W=$'\e[1;31;103m'			# W for Warning: highlighted text.
	declare -gr B=$'\e[1m'				# B for Bold.
	declare -gr R=$'\e[0m'				# R for Reset.

	# Display a warning to the user.
	clear

	# Give the user some instruction.
	cat <<-END


		${SPACER}


		In these instructions, when I ask you to press ${B}Y${R} (for Yes) or ${B}N${R} (for No),
		you may type these in either uppercase or lowercase.

		${B}Everything else is case sensitive!${R}


	END

	read -rp 'Press Enter to continue reading. '

	# Show a warning.
	cat <<-END


		${SPACER}


		${B}WARNING${R}

		I must be run only as instructed in the instructions for
		Manual Full System Encryption.

		You can find the instructions here;
		https://help.ubuntu.com/community/ManualFullSystemEncryption

		Do not use me otherwise, as it will wreak havoc on your system, leaving it
		unbootable, and it will probably also delete your data.

		Please confirm that you have read and followed the instructions before proceeding.


	END

	# Ask for confirmation.
	local ANSWER
	read -rp "Type ${B}Y${R} to proceed, or anything else to cancel, and press Enter: ${B}" ANSWER
	echo "${R}"

	# Terminate if required.
	if [[ "${ANSWER,}" != 'y' ]]
	then
		echo
		echo 'Terminated. I did nothing.'
		echo
		exit 1
	fi

	cat <<-END

		${SPACER}


		I shall ask you some questions.
		Please take care to answer them correctly.
		If you make a mistake while typing, press the Backspace key to rub out your answer.
		If you enter a mistake, you'll have to cancel and start again, sorry.

		You can cancel AT ANY TIME by pressing Ctrl+C.
		That is, you hold down the Ctrl button, then press the letter C,
		then let go of both keys.

		Warning: If you press Ctrl+C after I've already started working (I'll tell you when
		I start), your system might be left in an unbootable state.



	END

	read -rp 'Press Enter to proceed, or Ctrl+C to cancel. '

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


#===================================================================================================
#
#	Section: Gather Data
#
#===================================================================================================


#---------------------------------------------------------------------------------------------------
#	Gather all of the required data.
#
# Return
#	BOOTLOADER		/dev/[drive] e.g./dev/sda, /dev/nvme0n1
#	DATA_PARTITION_CHOSEN	true or false
#	SWAP_PARTITION_CHOSEN	true or false
#	SWAP_PARTITION_SIZE	Size in MiB or unset
#	HIBERNATION_CHOSEN	true or false
#	PARTITION_ESP		/dev/[partition] e.g. /dev/sda2, /dev/nvme0n1p2
#	PARTITION_SYSTEM	/dev/[partition] e.g. /dev/sda5, /dev/nvme0n1p5
#	PARTITION_DATA		/dev/[partition] e.g. /dev/sdb1, /dev/nvme0n2p1. Unset if unused.
#	PASSPHRASE_SYSTEM	Passphrase for the system partition
#	PASSPHRASE_DATA		Passphrase for the data partition or unset
#---------------------------------------------------------------------------------------------------


function gatherData ()
{
	findPartitions					# The various partitions.
	findBootloader					# The bootloader.
	findPassphrases					# The passphrases (and password note).

	# Prepare to give a summary.
	if ${DATA_PARTITION_CHOSEN}
	then
		local -r DATA_MSG="$(
			cat <<-END
				Partition ${B}${PARTITION_DATA}${R} will be used for your Data partition (/home).
				                      Passphrase: ${B}${PASSPHRASE_DATA}${R}
			END
				    )"
	else
		local -r DATA_MSG='You have chosen not to have a separate data partition.'
	fi
	if ${SWAP_PARTITION_CHOSEN}
	then
		local -r SWAP_MSG="$(
			cat <<-END
				Swap size ${B}${SWAP_PARTITION_SIZE}${R} will be created.
				                      Hibernation and, if possible, hybrid suspend will be enabled.
			END
				    )"
	else
		local -r SWAP_MSG="$(
			cat <<-'END'
				You have chosen not to have a separate swap partition.
				                      Neither hibernation nor hybrid suspend will be enabled.
			END
				    )"
	fi

	# Give a summary.
	cat <<-END


		${SPACER}


		${B}Summary${R}

		Partition ${B}${PARTITION_ESP}${R} will be used for the EFI System Partition.
		Partition ${B}${PARTITION_SYSTEM}${R} will be used for your System partition (root).
		                      Passphrase: ${B}${PASSPHRASE_SYSTEM}${R}
		${DATA_MSG}
		${SWAP_MSG}

		Your bootloader will be on drive ${B}${BOOTLOADER}${R}.


		${W}Please check the details carefully before deciding whether or not to proceed.${R}


		Are you sure that these details are correct?

	END

	# Confirm permission.
	local ANSWER=''
	read -rp "Type ${B}Y${R} to proceed, or anything else to cancel, and press Enter: ${B}" ANSWER
	echo "${R}"

	if [[ "${ANSWER,}" != 'y' ]]
	then
		echo
		echo 'Terminated. I did nothing.'
		echo
		exit 3					# Terminate if incorrect.
	fi

} # gatherData


#---------------------------------------------------------------------------------------------------
#	Find the partitions.
#
# Return
#	DATA_PARTITION_CHOSEN	true or false
#	SWAP_PARTITION_CHOSEN	true or false
#	SWAP_PARTITION_SIZE	Size in MiB or unset
#	HIBERNATION_CHOSEN	true or false
#	PARTITION_ESP		/dev/[partition] e.g. /dev/sda2, /dev/nvme0n1p2
#	PARTITION_SYSTEM	/dev/[partition] e.g. /dev/sda5, /dev/nvme0n1p5
#	PARTITION_DATA		/dev/[partition] e.g. /dev/sdb1, /dev/nvme0n2p1. Unset if unused.
#---------------------------------------------------------------------------------------------------


function findPartitions ()
{
	local ANSWER=''					# For user responses.

	# Do we have a swap partition?
	while true
	do
		cat <<-END

			${SPACER}

			The instructions asked you to choose whether or not to have a swap partition.
			Did you choose to have a swap partition?

			If you didn't choose to have a swap partition, just press Enter.
			-  Hibernation will be disabled.

			If you chose to have a swap partition, enter the size that you want in MiB (or MB is close enough).
			-  Hibernation and, if possible, hybrid suspend will be enabled.

		END
		read -rp "Enter the size in MiB, or leave blank for no swap: ${B}" ANSWER
		echo "${R}"

		[[ -z "${ANSWER}" ]] && break		# No swap.

		if [[ "${ANSWER}" =~ ^[1-9][0-9]{3,}$ ]]
		then
			ANSWER=$(( ${ANSWER} ))	# Convert to a number.
			break
		fi

		error 'The number must start with a non-zero and have at least 4 digits.'
	done
	if [[ -z ${ANSWER} ]]
	then
		declare -gr SWAP_PARTITION_CHOSEN=false
	else
		declare -gr SWAP_PARTITION_CHOSEN=true
		declare -gr SWAP_PARTITION_SIZE=${ANSWER}
	fi

	# For now, hibernation follows swap.
	declare -gr HIBERNATION_CHOSEN=${SWAP_PARTITION_CHOSEN}

	# Do we have a data partition?
	until [[ "${ANSWER,}" =~ ^[yn]$ ]]
	do
		cat <<-END

			${SPACER}

			The instructions asked you to choose whether or not to have a separate data partition.
			Did you choose to have a separate partition?

		END
		read -rp "Press ${B}Y${R} for Yes, or ${B}N${R} for No, and then Enter. ${B}" ANSWER
		echo "${R}"
	done
	if [[ ${ANSWER,} == y ]]
	then
		declare -gr DATA_PARTITION_CHOSEN=true
	else
		declare -gr DATA_PARTITION_CHOSEN=false
	fi

	# The ESP partition.
	while true
	do
		cat <<-END

			${SPACER}

			Which is your EFI System Partition (ESP)?
			Type just the part that goes after /dev/, no spaces, and press Enter.

		END
		read -rp "/dev/${B}" ANSWER
		echo "${R}"

		# Validate the partition.
		validatePartition "${ANSWER}" fat32 'EFI System Partition' false && break
	done
	declare -gr PARTITION_ESP=/dev/${ANSWER}

	# The system partition.
	while true
	do
		cat <<-END

			${SPACER}
			Which is your System partition?
			Type just the part that goes after /dev/, no spaces, and press Enter.

		END
		read -rp "/dev/${B}" ANSWER
		echo "${R}"

		# Validate the partition.
		validatePartition "${ANSWER}" cleared system true && break
	done
	declare -gr PARTITION_SYSTEM=/dev/${ANSWER}

	# The data partition.
	if ${DATA_PARTITION_CHOSEN}
	then
		while true
		do
			cat <<-END

				${SPACER}

				Which is your Data partition?
				Type just the part that goes after /dev/, no spaces, and press Enter.

			END
			read -rp "/dev/${B}" ANSWER
			echo "${R}"

			if [[ /dev/${ANSWER} == ${PARTITION_SYSTEM} ]]
			then
				error "That's the same as the System partition ${PARTITION_SYSTEM}. They can't be the same partition."
				continue
			fi

			# Validate the partition.
			validatePartition "${ANSWER}" cleared data true && break

		done

		declare -gr PARTITION_DATA=/dev/${ANSWER}
	fi

} # findPartitions


#---------------------------------------------------------------------------------------------------
#	Validate a partition.
#
# Parameter
#	1	The partition, e.g. sda2, nvme0n1p2
#	2	The required file system, either fat32 or cleared
#	3	The expected GPT label for the partition.
#	4	Whether or not to enforce a GPT label. true or false
#
# Return
#	0 if valid, non-zero otherwise.
#
# Output
#	An error message to &2 if invalid.
#---------------------------------------------------------------------------------------------------


function validatePartition ()
{
	local -r PARTITION="${1}"			# The partition.
	local -r REQUIRED_FS=${2}			# The required file system.
	local -r GPT_LABEL="${3}"			# The expected GPT label for the partition.
	local -r ENFORCE_LABEL=${4}			# Whether or not to enforce a label match.

	# Check the syntax.
	if ! [[ "${PARTITION}" =~ ^[a-z0-9]+$ ]]
	then
		error 'The syntax looks wrong. Please try again.'
		return 1
	fi

	#  Extract the drive.
	local DRIVE=$( readlink /sys/class/block/${PARTITION} )
	[[ -n ${DRIVE} ]] && DRIVE=${DRIVE%/*}
	[[ -n ${DRIVE} ]] && DRIVE=${DRIVE##*/}

	if [[ -z "${DRIVE}" ]]
	then
		error 'That drive or partition appears not to exist. Please try again. (code 1)'
		return 2
	fi

	# Find the specific partition. Needs sudo for some partitions only.
	local -r PARTITION_DETAILS=$( sudo blkid /dev/${PARTITION} 2>/dev/null )

	if [[ -z "${PARTITION_DETAILS}" ]]
	then
		error 'That partition appears not to exist. Please try again.'
		return 4
	fi

	# Extract the file system.
	local FILE_SYSTEM=$( grep -E --only-matching ' TYPE="[a-z0-9]+" ' <<<"${PARTITION_DETAILS}" | cut --delimiter=\" --field=2 )

	if [[ -z "${FILE_SYSTEM}" ]]
	then
		FILE_SYSTEM=cleared			# A cleared file system returns nothing.
	elif [[ ${FILE_SYSTEM} == vfat ]]
	then
		FILE_SYSTEM=fat32			# For now, assume FAT32, but we check later.
	fi

	# Check that the file system is correct.
	if [[ ${FILE_SYSTEM} != ${REQUIRED_FS} ]]
	then
		error "The file system should be ${REQUIRED_FS} but is instead ${FILE_SYSTEM}."
		return 5
	fi

	# For vfat, check that it really is FAT32 and not FAT16 or something else.
	if [[ ${REQUIRED_FS} == fat32 ]]
	then
		FAT32=$( sudo file --special-files /dev/${PARTITION} 2>/dev/null | grep -F --only-matching 'FAT (32 bit)' )
		if [[ -z ${FAT32} ]]
		then
			error "The file system should be ${REQUIRED_FS} but is instead a different vfat."
			return 6
		fi
	fi

	# Find the partition label.
	local -r PARTITION_LABEL="$( grep -E --only-matching ' PARTLABEL="[^\"]+" ' <<<"${PARTITION_DETAILS}" | cut --delimiter=\" --field=2 )"

	# If we don't have to enforce the label, do a case-insensitive check; otherwise check case.
	if ${ENFORCE_LABEL}
	then
		if [[ "${PARTITION_LABEL}" == "${GPT_LABEL}" ]]
		then
			local -r LABEL_MATCH=true
		else
			local -r LABEL_MATCH=false
		fi
	elif [[ "${PARTITION_LABEL,,}" == "${GPT_LABEL,,}" ]]
	then
		local -r LABEL_MATCH=true
	else
		local -r LABEL_MATCH=false
	fi

	# Check for a name discrepancy.
	if ! ${LABEL_MATCH}
	then
		# Tell the user of the discrepancy.
		if [[ -z "${PARTITION_LABEL}" ]]
		then
			error "The partition label should be \"${GPT_LABEL}\" but is instead blank."
		else
			error "The partition label should be \"${GPT_LABEL}\" but is instead \"${PARTITION_LABEL}\"."
		fi
		${ENFORCE_LABEL} && return 7		# Try again if we must enforce the label.

		# Ask the user whether or not to continue.
		local ANSWER
		read -rp "If this is anyway correct, press ${B}Y${R} to continue or anything else to try again. " ANSWER
		[[ "${ANSWER,}" != 'y' ]] && return 7	# Try again.
	fi

	return 0					# Happy to continue.

} # validatePartition


#---------------------------------------------------------------------------------------------------
#	Find the bootloader.
#
# Return
#	BOOTLOADER		/dev/[partition], e.g. /dev/nvme0n1
#---------------------------------------------------------------------------------------------------


function findBootloader ()
{
	local ANSWER					# User input.

	while true
	do
		cat <<-END

			${SPACER}

			On which drive are you going to put the bootloader?
			In other words, which drive are you booting from?
			If you are not sure, it is almost certainly where your EFI System Partition (ESP) is.
			Example:
			        Your ESP is on /dev/sda1
			        Your drive is probably /dev/sda
			Example:
			        Your ESP is on /dev/nvme0n1p2
			        Your drive is probably /dev/nvme0n1
			Type just the part that goes after /dev/, no spaces, and press Enter.

		END
		read -rp "/dev/${B}" ANSWER
		echo "${R}"

		validateBootloader "${ANSWER}" && break	# Validated.
	done

	declare -gr BOOTLOADER=/dev/${ANSWER}

} # findBootloader


#---------------------------------------------------------------------------------------------------
#	Validate the bootloader.
#
# Parameter
#	1	The drive, e.g. /dev/sda, /dev/nvme0n1
#
# Return
#	0 if valid, non-zero otherwise.
#
# Output
#	An error message to &2 if invalid.
#---------------------------------------------------------------------------------------------------


function validateBootloader ()
{
	local -r DRIVE="${1}"

	# Check the syntax.
	if ! [[ "${DRIVE}" =~ ^[a-z0-9]{3,}$ ]]
	then
		error 'The syntax is wrong. It should be something like /dev/sda or /dev/nvme0n1.'
		return 1
	fi

	# Find the drive's details.
	local -r DRIVE_DETAILS="$( sudo fdisk --list /dev/${DRIVE} 2>/dev/null )"

	if [[ -z "${DRIVE_DETAILS}" ]]
	then
		error 'That drive appears not to exist. Please try again.'
		return 2
	fi

	# Find the disk label.
	local -r TABLE_TYPE="$( grep -F 'Disklabel type: ' <<<"${DRIVE_DETAILS}" | cut --delimiter=' ' --fields=3 )"
	if [[ -z ${TABLE_TYPE} ]]
	then
		error "I'm having trouble finding the drive's partition table.\nAre you sure that it's /dev/${DRIVE}?"
		return 3
	fi
	if [[ "${TABLE_TYPE}" != 'gpt' ]]
	then
		error "That drive needs a 'gpt' partition table, but its table is of type '${TABLE_TYPE}'."
		return 4
	fi

} # validateBootloader


#---------------------------------------------------------------------------------------------------
#	Find the passphrases.
#
# Return
#	PASSPHRASE_SYSTEM
#	PASSPHRASE_DATA		Only if required otherwise unset
#---------------------------------------------------------------------------------------------------


function findPassphrases ()
{
	# Read the system passphrase.
	local SYSTEM=''
	while true
	do
		cat <<-END

			${SPACER}

			Please enter the passphrase that you chose for your system partition.
			Check carefully before you press Enter.

		END
		read -rp "System passphrase: ${B}" SYSTEM
		echo "${R}"

		(( ${#SYSTEM} > 0 )) && break
	done
	declare -gr PASSPHRASE_SYSTEM="${SYSTEM}"

	# Read the data passphrase.
	if ${DATA_PARTITION_CHOSEN}
	then
		local DATA=''
		while true
		do
			cat <<-END

				${SPACER}

				Please enter the passphrase that you chose for your data partition.
				Check carefully before you press Enter.

			END
			read -rp "Data passphrase: ${B}" DATA
			echo "${R}"

			if (( ${#DATA} > 0 ))
			then
				if [[ "${SYSTEM}" == "${DATA}" ]]
				then
					error 'Your system and data passphrases are the same. They must be different.'
				else
					break
				fi
			fi
		done
		declare -gr PASSPHRASE_DATA="${DATA}"
	fi

	# Let the user know about the password.
	echo
	echo "I shan't ask for your Ubuntu login password now, because the Installer will do so later."
	echo
	read -rp 'Press Enter to continue. '

} # findPassphrases


#===================================================================================================
#
#	Section: Pre-installation Process
#
#===================================================================================================


#---------------------------------------------------------------------------------------------------
#	Pre-installation process
#---------------------------------------------------------------------------------------------------


function preInstallationProcess ()
{
	# Give the user a final warning.
	cat <<-END


		${SPACER}


		I am about to start working.
		If you press Ctrl+C now, I won't have done anything.
		But if you press Ctrl+C after I start working, you will have to restart the installation.

	END

	# Pause while the user decides.
	read -rp 'Press Enter to let me start working, or press Ctrl+C to cancel: '

	# Encrypt the system partition.
	encryptPartition System ${PARTITION_SYSTEM} "${PASSPHRASE_SYSTEM}"

	# Unlock the system partition.
	unlockPartition System system ${PARTITION_SYSTEM} "${PASSPHRASE_SYSTEM}"

	setUpLvm System system				# Set up the system LVM.

	setUpLogicalVolume Boot boot system 512M	# Create /boot.
	formatVolume Boot system-boot ext4 boot		# Format /boot.

	if ${SWAP_PARTITION_CHOSEN}
	then
		# Create swap.
		setUpLogicalVolume Swap swap system ${SWAP_PARTITION_SIZE}M
		formatVolume Swap system-swap swap swap	# Format /boot.
	fi

	setUpLogicalVolume Root root system '100%FREE'	# Create root.
	formatVolume Root system-root ext4 root		# Format root.

	if ${DATA_PARTITION_CHOSEN}
	then
		# Encrypt the data partition.
		encryptPartition Data ${PARTITION_DATA} "${PASSPHRASE_DATA}"

		# Unlock the data partition.
		unlockPartition Data data ${PARTITION_DATA} "${PASSPHRASE_DATA}"

		setUpLvm Data data			# Set up the data LVM.

		# Create /home.
		setUpLogicalVolume Data home data '100%FREE'
		formatVolume Data data-home ext4 home	# Format /home.
	fi

} # preInstallationProcess


#---------------------------------------------------------------------------------------------------
#	Encrypt a partition
#
# Parameters
#	1	Human-readable name for the partition
#	2	The partition, e.g. /dev/sda2, nvme0n1p2
#	3	The passphrase
#---------------------------------------------------------------------------------------------------


function encryptPartition ()
{
	local -r HUMAN_NAME=${1}
	local -r PARTITION=${2}
	local -r PASSPHRASE="${3}"

	echo
	echo "Encrypting the ${HUMAN_NAME} partition..."

	# Encrypt the partition.
	echo -n "${PASSPHRASE}" | sudo cryptsetup luksFormat --hash=sha512 --key-size=512 --key-file=- ${PARTITION}

	local -ir RET=${?}				# Catch the return code.

	(( RET )) && error "There was an error encrypting the ${HUMAN_NAME} partition." ${RET}

} # encryptPartition


#---------------------------------------------------------------------------------------------------
#	Unlock a partition
#
# Parameters
#	1	Human-readable name for the partition
#	2	Partition label
#	3	The partition, e.g. /dev/sda2, /dev/nvme0n1p2
#	4	The passphrase
#---------------------------------------------------------------------------------------------------


function unlockPartition ()
{
	local -r HUMAN_NAME=${1}
	local -r LABEL=${2}
	local -r PARTITION=${3}
	local -r PASSPHRASE="${4}"

	echo
	echo "Unlocking the ${HUMAN_NAME} partition..."

	# Unlock the partition.
	echo -n "${PASSPHRASE}" | sudo cryptsetup open --type=luks --key-file=- ${PARTITION} ${LABEL}

	local -ir RET=${?}				# Catch the return code.

	(( RET )) && error "There was an error unlocking the ${HUMAN_NAME} partition." ${RET}

} # unlockPartition


#---------------------------------------------------------------------------------------------------
#	Set up LVM for the partition.
#
# Parameters
#	1	Human-readable name for the partition
#	2	Partition label
#---------------------------------------------------------------------------------------------------


function setUpLvm ()
{
	local -r HUMAN_NAME=${1}
	local -r LABEL=${2}

	echo
	echo "Set up ${HUMAN_NAME} physical volume for ${LABEL}..."

	sudo pvcreate /dev/mapper/${LABEL}		# Initialise the physical volume.

	local -i RET=${?}				# Catch the return code.

	(( RET )) && error "There was an error initialising the physical volume for LVM on the ${HUMAN_NAME} partition." ${RET}

	echo
	echo "Set up ${HUMAN_NAME} volume group for ${LABEL}..."

	sudo vgcreate ${LABEL} /dev/mapper/${LABEL}	# Set up the volume group.

	RET=${?}					# Catch the return code.

	(( RET )) && error "There was an error setting up the volume group for LVM on the ${HUMAN_NAME} partition." ${RET}

} # setUpLvm


#---------------------------------------------------------------------------------------------------
#	Set up the logical volume for a partition.
#
# Parameters
#	1	Human-readable name for the partition
#	2	Partition label
#	3	Partition to set up
#	4	Size, including the modifier, e.g. 512M and 100%FREE
#---------------------------------------------------------------------------------------------------


function setUpLogicalVolume ()
{
	local -r HUMAN_NAME=${1}			# Human-readable name.
	local -r LABEL=${2}				# The logical volumne name.
	local -r PARTITION=${3}				# The partition where to create it.
	local -r SIZE=${4}				# The required size.

	echo
	echo "Set up logical volume ${HUMAN_NAME} for ${LABEL} in ${PARTITION} size ${SIZE}..."

	if [[ ${SIZE} == '100%FREE' ]]
	then
		local -r OPTION=extents
	else
		local -r OPTION=size
	fi

	sudo lvcreate --${OPTION}=${SIZE} --name=${LABEL} ${PARTITION}

	local -i RET=${?}				# Catch the return code.

	(( RET )) && error "There was an error initialising the logical volume for LVM on the ${HUMAN_NAME} partition." ${RET}

} # setUpLogicalVolume


#---------------------------------------------------------------------------------------------------
#	Format a volume.
#
# Parameters
#	1	Human-readable name for the partition
#	2	Partition to be formatted
#	3	Type of format, specifically swap or ext4
#	4	Lbel for the partition
#---------------------------------------------------------------------------------------------------


function formatVolume ()
{
	local -r HUMAN_NAME="${1}"
	local -r PARTITION=${2}
	local -r TYPE=${3}
	local -r LABEL="${4}"

	echo
	echo "Format ${HUMAN_NAME} partition ${PARTITION} (${LABEL}) as ${TYPE}..."

	# Format the partition.
	if [[ ${TYPE} == 'swap' ]]
	then
		sudo mkswap --label=${LABEL} /dev/mapper/${PARTITION}
	else
		sudo mkfs.ext4 -L ${LABEL} /dev/mapper/${PARTITION}
	fi

	local -ir RET=${?}				# Catch the return code.

	(( RET )) && error "Error formatting the ${HUMAN_NAME} partition in ${PARTITION}." ${RET}

} # formatVolume


#===================================================================================================
#
#	Section: Run the installer
#
#===================================================================================================


#---------------------------------------------------------------------------------------------------
#	Run the installer
#---------------------------------------------------------------------------------------------------


function runInstaller ()
{
	# Leave an explanation for the user.
	cat <<-END



		${SPACER}


		${W}Do NOT close this window!${R}

		I have finished preparing the system for the Ubuntu Installer.

		When you press Enter, I shall start the Installer (it may take a few moments to start).

		You should already have the instructions for the Installer open in your browser,
		but if you don't, here is the link:

		https://help.ubuntu.com/community/ManualFullSystemEncryption/DetailedProcessInstallUbuntu

		${B}When the Installer has finished, return here to this terminal to continue.${R}

		${W}Do NOT close this window!${R}

	END

	read -rp 'Press Enter to start the Installer. '

	# Leave a message for the user.
	echo
	echo 'Starting the Installer...'

	# Run this in the background, because on Mint, it hangs at the end.
	sudo --preserve-env=DBUS_SESSION_BUS_ADDRESS,XDG_RUNTIME_DIR sh -c 'ubiquity gtk_ui' &
	local -ir INSTALLER_PID=${!}			# Save the process ID.

	sleep 30s					# Wait for the installer to start.

	# Inform the user.
	cat <<-END


		${SPACER}


		${B}If the Installer did not start by itself, please start it manually
		(but don't close this window).${R}


		${B}When the Installer has finished, come back here.${R}

	END

	# Wait for the installer to finish.
	local -ar DOTS=( '.   ' ' .  ' '  . ' '   .' )	# For an animated display.
	local -i COUNT=3				# Toggle between 0 and 3 inclusive.
	while true
	do
		# Break when the process has ended.
		if ! [[ -e /proc/${INSTALLER_PID} ]]
		then
			echo -e $'\e[2K'		# Erase the line.
			break				# Break.
		fi

		# Prompt, because the Installer on some systems (e.g. Mint) hangs when it finishes.
		# Time out to allow repeat check if the Installer process has finished.
		(( ++COUNT > 3 )) && (( COUNT = 0 ))
		read -t 1 -rp "${B}Waiting for the Installer. Press Enter if it has already finished${DOTS[COUNT]}${R} "

		(( ${?} )) || break			# Break when the user presses Enter.

		echo -en '\r'				# Return to the start of the previous line.
	done

	cat <<-'END'

		The installer appears to have finished.
			If it didn't even start, it might have crashed. In that case,
			please start it manually, and return here once it has finished.

	END

} # runInstaller


#===================================================================================================
#
#	Section: Post-installation Process
#
#===================================================================================================


#---------------------------------------------------------------------------------------------------
#	Post-installation process
#---------------------------------------------------------------------------------------------------


function postInstallationProcess ()
{
	# Leave a friendly message.
	cat <<-END



		${SPACER}


		${B}Continue from here!${R}

		Did the Installer complete as per the instructions?
		If not, you should cancel (press Ctrl+C) and ask for help.

	END

	read -rp 'Otherwise, press Enter to continue. '
	echo

	mountPartitions					# Mount the partitions to prepare.
	receiveScripts					# Download required scripts.
	addEncryptionKeys				# Add keys to the encrypted partitions.
	setUpEncryptionAccess				# Set up the encryption access.

	if ${HIBERNATION_CHOSEN}
	then
		allowInstallation			# Add Universe and Multiverse.
		enableHibernation			# Enable hibernation and hybrid suspend.
	fi

	setDefaultGrub					# Set up Grub correctly.
	fixEfi						# Fix the EFI boot process.
	setUpDecryption					# Set up the decryption file.
	processChroot					# Enter chroot and continue.
	removeTemporaryFiles				# Remove any temporary files.

} # postInstallationProcess


#---------------------------------------------------------------------------------------------------
#	Mount the partitions to prepare for fixing the system.
#---------------------------------------------------------------------------------------------------


function mountPartitions ()
{
	local -i RET					# To catch errors.

	sudo mkdir /mnt/root				# Create a mount point for root.
	RET=${?}
	(( RET )) && error 'Error creating a mount point for root.' ${RET}

	sudo mount /dev/mapper/system-root /mnt/root	# Mount root.
	RET=${?}
	(( RET )) && error 'Error mounting root.' ${RET}

	# Mount /boot.
	sudo mount /dev/mapper/system-boot /mnt/root/boot
	RET=${?}
	(( RET )) && error 'Error mounting /boot.' ${RET}

	sudo mount ${PARTITION_ESP} /mnt/root/boot/efi	# Mount the EFI.
	RET=${?}
	(( RET )) && error 'Error mounting the EFI System Partition (ESP).' ${RET}

	if ${DATA_PARTITION_CHOSEN}
	then
		# Mount /home.
		sudo mount /dev/mapper/data-home /mnt/root/home
		RET=${?}
		(( RET )) && error 'Error mounting /home.' ${RET}
	fi

} # mountPartitions


#---------------------------------------------------------------------------------------------------
#	Download required scripts for later.
#---------------------------------------------------------------------------------------------------


function receiveScripts ()
{
	local -i RET					# To catch errors.

	# Receive the script for use after chroot.
	sudo wget --no-verbose --output-document=/mnt/root/usr/local/sbin/encryptinstallationchroot	\
		  https://www.dropbox.com/s/fx5przc19rp1n7d/encryptinstallationchroot?dl=1
	RET=${?}
	(( RET )) && error 'Unable to download the script encryptinstallationchroot.' ${RET}

	# Receive the script to refresh Grub.
	sudo wget --no-verbose --output-document=/mnt/root/usr/local/sbin/refreshgrub	\
		  https://www.dropbox.com/s/npoazngcj3khcvf/refreshgrub?dl=1
	RET=${?}
	(( RET )) && error 'Unable to download the script refreshgrub.' ${RET}

	sudo sed --in-place --expression "s^/dev/\[DRIVE\]^${BOOTLOADER}^" /mnt/root/usr/local/sbin/refreshgrub
	RET=${?}
	(( RET )) && error 'Unable to set the bootloader in refreshgrub.' ${RET}

	sudo chmod u+x /mnt/root/usr/local/sbin/encryptinstallationchroot /mnt/root/usr/local/sbin/refreshgrub
	RET=${?}
	(( RET )) && error 'Unable to allow running of encryptinstallationchroot and refreshgrub.' ${RET}

} # receiveScripts


#---------------------------------------------------------------------------------------------------
#	Add the encryption keys to the encrypted partitions.
#---------------------------------------------------------------------------------------------------


function addEncryptionKeys ()
{
	local -i RET					# To catch errors.

	# Create an encryption key for system.
	sudo dd if=/dev/urandom of=/mnt/root/etc/crypt.system count=1 bs=512
	RET=${?}
	(( RET )) && error 'Error creating the system encryption key.' ${RET}

	# Add the key to system.
	echo -n "${PASSPHRASE_SYSTEM}" | sudo cryptsetup luksAddKey --key-file=- ${PARTITION_SYSTEM} /mnt/root/etc/crypt.system
	RET=${?}
	(( RET )) && error 'Error adding the system encryption key.' ${RET}

	if ${DATA_PARTITION_CHOSEN}
	then
		# Create an encryption key for data.
		sudo dd if=/dev/urandom of=/mnt/root/etc/crypt.data count=1 bs=512
		RET=${?}
		(( RET )) && error 'Error creating the data encryption key.' ${RET}

		# Add the key to data.
		echo -n "${PASSPHRASE_DATA}" | sudo cryptsetup luksAddKey --key-file=- ${PARTITION_DATA} /mnt/root/etc/crypt.data
		RET=${?}
		(( RET )) && error 'Error adding the data encryption key.' ${RET}
	fi

} # addEncryptionKeys


#---------------------------------------------------------------------------------------------------
#	Set up access to the encrypted partitions.
#---------------------------------------------------------------------------------------------------


function setUpEncryptionAccess ()
{
	local -i RET					# To catch errors.

	# Find the UUID of the system's physical partition.
	local -r UUID_SYSTEM=$(
		lsblk --paths --output=NAME,UUID --noheadings ${PARTITION_SYSTEM}	|
			grep -E "^${PARTITION_SYSTEM} "					|
			tr --squeeze-repeats ' '					|
			cut --delimiter=' ' --field=2
			      )
	RET=${?}
	if (( RET )) || [[ -z ${UUID_SYSTEM} ]]
	then
		error "Error finding the UUID for the System partition ${PARTITION_SYSTEM}." ${RET}
	fi

	if ${DATA_PARTITION_CHOSEN}
	then
		# Find the UUID of the data's physical partition.
		local -r UUID_DATA=$(
			lsblk --paths --output=NAME,UUID --noheadings ${PARTITION_DATA}	|
				grep -E "^${PARTITION_DATA} "				|
				tr --squeeze-repeats ' '				|
				cut --delimiter=' ' --field=2
				    )
		RET=${?}
		if (( RET )) || [[ -z ${UUID_DATA} ]]
		then
			error "Error finding the UUID for the Data partition ${PARTITION_DATA}." ${RET}
		fi
	fi

	# Create the crypttab for system.
	sudo tee /mnt/root/etc/crypttab >/dev/null	\
		<<<"$(
			cat <<-END
				#name> <source device>                           <key file>        <options>
				system UUID=${UUID_SYSTEM} /etc/crypt.system luks,discard,noearly,keyscript=/lib/cryptsetup/scripts/getinitramfskey.sh
			END
		     )"
	RET=${?}
	(( RET )) && error 'Error setting up crypttab for system.' ${RET}

	# Add data if required.
	if ${DATA_PARTITION_CHOSEN}
	then
		sudo tee --append /mnt/root/etc/crypttab >/dev/null	\
			<<<"$(
				cat <<-END
					data   UUID=${UUID_DATA} /etc/crypt.data   luks,discard,noearly
				END
			     )"
		RET=${?}
		(( RET )) && error 'Error setting up crypttab for data.' ${RET}
	fi

	sudo chmod -rw /mnt/root/etc/crypt*		# Improve security.
	RET=${?}
	(( RET )) && error 'Error changing crypt* permissions.' ${RET}

} # setUpEncryptionAccess


#---------------------------------------------------------------------------------------------------
#	Add Universe and Multiverse to the repositories.
#---------------------------------------------------------------------------------------------------


function allowInstallation ()
{
	local -i RET					# To catch errors.

	# Add Universe and Multiverse into the allowed APT sources.
	sudo sed --in-place=.backup						\
		 --expression='/^deb http/{/multiverse/! s/$/ multiverse/}'	\
		 --expression='/^deb http/{/universe/! s/$/ universe/}'		\
		 /etc/apt/sources.list
	RET=${?}
	if (( RET ))
	then
		error 'Error adding unverse and multiverse into /etc/apt/sources.list (Live CD). (Continuing with the next step.)'
		return ${RET}
	fi

	sudo apt update --quiet				# Update the repositories.
	RET=${?}
	if (( RET ))
	then
		error 'Error updating repositories with unverse and multiverse. (Continuing with the next step.)'
		return
	fi

} # allowInstallation


#---------------------------------------------------------------------------------------------------
#	Enable hibernation and hybrid suspend.
#---------------------------------------------------------------------------------------------------


function enableHibernation ()
{
	# Enable hibernation.
	sudo tee /mnt/root/etc/polkit-1/localauthority/50-local.d/com.ubuntu.enable-hibernate.pkla >/dev/null	\
		<<<"$(
			cat <<-'END'
				[Re-enable hibernate by default in upower]
				Identity=unix-user:*
				Action=org.freedesktop.upower.hibernate
				ResultActive=yes

				[Re-enable hibernate by default in logind]
				Identity=unix-user:*
				Action=org.freedesktop.login1.hibernate;org.freedesktop.login1.hibernate-multiple-sessions
				ResultActive=yes
			END
		     )"
	if (( ${?} ))
	then
		error 'Error enabling hibernation. (Continuing with the next step.)'
		return
	fi

	# Repair resume from hibernation.
	echo 'RESUME=/dev/mapper/system-swap' | sudo tee /mnt/root/etc/initramfs-tools/conf.d/resume >/dev/null
	if (( ${?} ))
	then
		error 'Error repairing resume from hibernation. (Continuing with the next step.)'
		return
	fi

	# Install the required application.
	sudo apt install --assume-yes --quiet pm-utils 2>/dev/null
	if (( ${?} ))
	then
		error 'I cannot enable hybrid syspend. (Continuing with the next step.)'
		return
	fi

	# Check if the hardware supports hybrid suspend.
	local -r HS_ENABLED=$( sudo pm-is-supported --suspend-hybrid && echo 'true' || echo 'false' )
	if ! ${HS_ENABLED}
	then
		error 'Hybrid suspend is not possible on this hardware. (Continuing with the next step.)'
		return
	fi

	# Enable hybrid syspend.
	sudo tee --append /mnt/root/etc/systemd/logind.conf >/dev/null	\
		<<<"$(
			cat <<-'END'
				HandleSuspendKey=hybrid-sleep
				HandleLidSwitch=hybrid-sleep
			END
		     )"
	if (( ${?} ))
	then
		error 'Error enabling hybrid syspend. (Continuing with the next step.)'
		return
	fi

} # enableHibernation


#---------------------------------------------------------------------------------------------------
#	Set up Grub correctly.
#---------------------------------------------------------------------------------------------------


function setDefaultGrub ()
{
	local -i RET					# To catch errors.

	# Display the menu.
	sudo sed --in-place --expression='/GRUB_TIMEOUT_STYLE=hidden/ s/hidden/menu/' /mnt/root/etc/default/grub
	(( ${?} )) && error 'Unable to display the Grub menu. (Continuing with the next step.)'

	# Enable cryptography.
	echo 'GRUB_ENABLE_CRYPTODISK=y' | sudo tee --append /mnt/root/etc/default/grub >/dev/null
	RET=${?}
	(( RET )) && error 'Unable to enable cryptography in Grub.' ${RET}

} # setDefaultGrub


#---------------------------------------------------------------------------------------------------
#	Fix the EFI boot process.
#---------------------------------------------------------------------------------------------------


function fixEfi ()
{
	# Create the EFI boot instruction.
	echo '\EFI\ubuntu\grubx64.efi' | sudo tee /mnt/root/boot/efi/startup.nsh >/dev/null
	local -ir RET=${?}
	(( RET )) && error 'Unable to create the EFI boot instruction.' ${RET}

} # fixEfi


#---------------------------------------------------------------------------------------------------
#	Set up the decryption file.
#---------------------------------------------------------------------------------------------------


function setUpDecryption ()
{
	local -i RET=${?}				# To catch errors.

	# Obtain the decryption.
	sudo tee /mnt/root/lib/cryptsetup/scripts/getinitramfskey.sh >/dev/null	\
		<<<"$(
			cat <<-'END'
				# File:
				#       /lib/cryptsetup/scripts/getinitramfskey.sh
				#
				# Description:
				#       Called by initramfs using busybox ash to obtain the decryption key for the system.
				#
				# Purpose:
				#       Used with loadinitramfskey.sh in full disk encryption to decrypt the system LUKS partition,
				#       to prevent being asked twice for the same passphrase.

				KEY="${1}"

				if [ -f "${KEY}" ]
				then
				        cat "${KEY}"
				else
				        PASS=/bin/plymouth ask-for-password --prompt="Key not found. Enter LUKS Password: "
				        echo "${PASS}"
				fi

				#<<EOF
			END
		     )"
	RET=${?}
	(( RET )) && error 'Unable to create the initramfs "obtain decryption" file.' ${RET}

	# Use the decryption file.
	sudo tee /mnt/root/etc/initramfs-tools/hooks/loadinitramfskey.sh >/dev/null	\
		<<<"$(
			cat <<-'END'
				# File:
				#       /etc/initramfs-tools/hooks/loadinitramfskey.sh
				#
				# Description:
				#       Called by update-initramfs and loads getinitramfskey.sh to obtain the system decryption key.
				#
				# Purpose:
				#       Used with getinitramfskey.sh in full disk encryption to decrypt the system LUKS partition,
				#       to prevent being asked twice for the same passphrase.

				PREREQ=""

				prereqs()
				{
				        echo "${PREREQ}"
				}

				case "${1}" in
        				prereqs)
				                prereqs
				                exit 0
				        ;;
				esac

				. "${CONFDIR}"/initramfs.conf

				. /usr/share/initramfs-tools/hook-functions

				if [ ! -f "${DESTDIR}"/lib/cryptsetup/scripts/getinitramfskey.sh ]
				then
        				if [ ! -d "${DESTDIR}"/lib/cryptsetup/scripts/ ]
				        then
				                mkdir --parents "${DESTDIR}"/lib/cryptsetup/scripts/
				        fi
				        cp /lib/cryptsetup/scripts/getinitramfskey.sh "${DESTDIR}"/lib/cryptsetup/scripts/
				fi

				if [ ! -d "${DESTDIR}"/etc/ ]
				then
				        mkdir -p "${DESTDIR}"/etc/
				fi

				cp /etc/crypt.system "${DESTDIR}"/etc/

				#<<EOF
			END
		     )"
	RET=${?}
	(( RET )) && error 'Unable to create the initramfs "use decryption" file.' ${RET}

	sudo chmod +x /mnt/root/lib/cryptsetup/scripts/getinitramfskey.sh /mnt/root/etc/initramfs-tools/hooks/loadinitramfskey.sh

} # setUpDecryption


#---------------------------------------------------------------------------------------------------
#	Enter chroot and continue within it.
#---------------------------------------------------------------------------------------------------


function processChroot ()
{
	local -i RET					# To catch errors.

	sudo mount --bind /dev /mnt/root/dev		# Mount dev in preparation for chroot.
	RET=${?}
	(( RET )) && error 'Unable to mount dev for chroot.' ${RET}

	sudo mount --bind /run /mnt/root/run		# Mount run in preparation for chroot.
	RET=${?}
	(( RET )) && error 'Unable to mount run for chroot.' ${RET}

	# Begin chroot and continue within it; wait until it has finished before continuing.
	sudo chroot /mnt/root encryptinstallationchroot ${HIBERNATION_CHOSEN}
	RET=${?}
	(( RET )) && error 'chroot returned an error.' ${RET}

} # processChroot


#---------------------------------------------------------------------------------------------------
#	Remove any temporary files.
#---------------------------------------------------------------------------------------------------


function removeTemporaryFiles ()
{
	# Remove the chroot file.
	sudo rm /mnt/root/usr/local/sbin/encryptinstallationchroot
	local -ir RET=${?}
	(( RET )) && error 'Unable to remove temporary file encryptinstallationchroot. (Continuing with the next step.)'

} # removeTemporaryFiles


#===================================================================================================
#
#	Section: Finish up the script.
#
#===================================================================================================


#---------------------------------------------------------------------------------------------------
#	Let the user know what to do next.
#---------------------------------------------------------------------------------------------------


function finalise ()
{
	cat <<-END



		${SPACER}



		${W}The installation process is complete!${R}

		Thank you for your patience.

		${B}Please return to instructions in your browser and continue from where you left off.${R}
		In case you have lost your place, resume here:

		https://help.ubuntu.com/community/ManualFullSystemEncryption/DetailedProcessCheckAndFinalise

		You may close this terminal window at any time.
	END


} # finalise


#===================================================================================================
#
#	Section: Main script control.
#
#===================================================================================================


####################################################################################################
#	MAIN SCRIPT CONTROL
####################################################################################################


initialise						# Set up the script.
gatherData						# Gather all the required data.
preInstallationProcess					# Do the pre-installation work.
runInstaller						# Run the installer.
postInstallationProcess					# Do the post-installation work.
finalise						# Finish up the script.
