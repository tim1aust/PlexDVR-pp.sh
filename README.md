#!/bin/sh
#set -x

# -- Plex DVR Post Processing, requires: comskip, comcut & HandBrakeCLI

# Use the following Handbrake standard preset - change as needed
#				for a list, see: HandBrakeCLI -z

HB_PRESET="H.264 MKV 1080p30"		# default preset (but see .hb_preset below)

INAME="/home/XXX/plex-comskip.ini"		# comskip .ini
LOCKF="/mnt/work/video/comcut/lock"		# comcut lockfile

# Comment the following if you don't want/have mail delivered/configured
#				or, change user(s) as needed

MAILUSER="root@localhost"
MAILUSR2="XXX@gmail.com"		# (postfix works nicely for me)

if [ $# -gt 1 ]; then		# allow '--skip' arg (to skip commercials)
	skip="y"
	[ "$1" = "--skip" ] && shift 1 || shift $#
fi

skip="y"		# nevermind, always use comskip/comcut..

if [ $# -ne 1 -o ! -f "$1" ]; then
	echo "usage: $0 [--skip] <path-to-video-file>"	# input filepath is required lone argument
	exit 0
fi
   
FILENAME=$1 	# input file
	
FNAME=`basename "$FILENAME"`	# strip path
TITLE=${FNAME%.ts}				# just the episode Title (?)

HDR="-- TITLE --\n$TITLE\n-----------"

if [ "$FNAME" = "$TITLE" ]; then		# not a transport stream (*.ts) input file

	MSG="$HDR\nNo transcoding - not a video transport stream (*.ts) input."
else
	if [ -n "$skip" ]; then		# skip commercials first (if asked)

		PNAME=${FILENAME%/*}		# FILENAME's pathname

		/bin/comskip "--ini=${INAME}" \
			"--output=${PNAME}" -d 255 -t "${FILENAME}" "${PNAME}" >/tmp/comskip.log 2>&1

		ls -alt "${PNAME}" >/tmp/com-ls.log		# save some logs: "/tmp/*.log"

		# cut commercials -- must have output_edl=1 in .ini file!
		"/usr/local/bin/comcut" \
			--keep-edl \
			--ffmpeg=/bin/ffmpeg \
			--comskip=/bin/comskip \
			--comskip-ini="${INAME}" \
			--lockfile="${LOCKF}" \
			"${FILENAME}" >/tmp/comcut.log 2>&1
			
	fi

	# Allow for an HB_PRESET override (one per TV show/series), in a file named .hb_preset
	#		If <TV-series-directory>/.hb_preset exists, it should have one line: HB_PRESET="xxx"
	  
	TVDIR=${FILENAME%%.grab*}		# plex directory for TV Shows
	PGMDIR="$TVDIR${TITLE%% - *}"	# plex directory for the current TV Show/series

	[ -f "$PGMDIR/.hb_preset" ] && . "$PGMDIR/.hb_preset"		
	
	echo "========================================================"
	echo "Transcoding with HandBrakeCLI preset '"$HB_PRESET"'" 
	echo "========================================================"

	[ -z "${HB_PRESET##* MKV *}" -o -z "${HB_PRESET##* 4K *}" ] && EXT="mkv" || EXT="mp4"

	SAVENAME="${FILENAME%.ts}.$EXT"  # output file: new extension, same directory
		
	SZ1=`du -hs "$FILENAME" | cut -f1`	# original filesize

	SECONDS=0
	LD_LIBRARY_PATH="/usr/lib:"$LD_LIBRARY_PATH HandBrakeCLI -i "$FILENAME" \
		--preset "$HB_PRESET" -o "$SAVENAME" >/tmp/com-hb.log 2>&1
	CC=$?

	if [ $CC -ne 0 ]; then		# leave any partial output file

		MSG="$HDR\nTranscoding failed, CC=$CC."
	else
		XTIME=$SECONDS		# transcoding time

		rm -f "$FILENAME"		# remove original bulky file
	
		SZ2=`du -hs "$SAVENAME" | cut -f1`	# transcoded filesize

		MSG="$HDR\nFilesize reduced from $SZ1 to $SZ2, in $XTIME seconds (using '$HB_PRESET')"
	fi
fi

echo "========================================================"
echo -e "$MSG"
echo "========================================================"

if [ -n "$MAILUSER" ]; then	# mail requested

	if [ ! -f "$HOME/.rnd" ]; then	# ensure $HOME/.rnd exists
		echo "========================================================"
		echo "mail not sent - $HOME/.rnd is missing"
		echo "try:"
		echo "	sudo touch $HOME/.rnd"
		echo "	sudo chown $USER:$USER $HOME/.rnd"
		echo "	sudo chmod 600 $HOME/.rnd"
		echo "========================================================"
	else
		dd if=/dev/urandom of=$HOME/.rnd bs=256 count=1 >/dev/null 2>&1

		echo -e "$MSG" | /bin/mail -Stypescript-mode -Snosendwait -s "Plex transcode" $MAILUSER

		# 2nd mail recipient ?
		[ -n "${MAILUSR2}" ] && \
			echo -e "$MSG" | /bin/mail -Stypescript-mode -Snosendwait -s "Plex transcode" ${MAILUSR2}
	fi
fi
