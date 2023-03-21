#!/bin/sh
SCRIPTVERSION="2.61"
#this is the PROD release for TSC helper script
BETA=false
SCRIPTURL="https://raw.githubusercontent.com/ToonSoftwareCollective/tscSettings/main"
RESOURCESURL="https://raw.githubusercontent.com/ToonSoftwareCollective/resourcefiles/main"


# load a sane environment
. /etc/profile

ROUNDWAIT=5
BETACHECKTIME=0
UPDATECHECKINTERVAL=43200
STARTTIME=`date +%s`
LASTTIMEWLPM=`date +%s` 
LASTUPDATECHECK=0
COMMANDFILE=/tmp/tsc.command
NOTIFYUPDATE=true


checkSelfUpdate() {
	CURRENTTIME=`date +%s`
	MD5TSCONLINE=`curl -Nks -A "$HOSTNAME:$SCRIPTVERSION" $SCRIPTURL/tsc.md5?$CURRENTTIME | cut -d\  -f1`
	MD5TSCNOW=`/usr/bin/md5sum /usr/bin/tsc | cut -d\  -f1`
	if [ !  "$MD5TSCNOW" == "$MD5TSCONLINE" ] && [ -n "$MD5TSCONLINE" ]
	then
		echo "$(date '+%d/%m/%Y %H:%M:%S') There is a new version of the TSC script available. Updating and restarting!"
		CHECKRESULT=true
		/usr/bin/curl -Nks $SCRIPTURL/tsc -o /usr/bin/tsc.new
		MD5TSCNOW=`/usr/bin/md5sum /usr/bin/tsc.new | cut -d\  -f1`
		if [ "$MD5TSCNOW" == "$MD5TSCONLINE" ]
		then
			echo "$(date '+%d/%m/%Y %H:%M:%S') Download ok! Replacing myself and restarting."
			mv /usr/bin/tsc.new /usr/bin/tsc
			chmod +x /usr/bin/tsc
			#remove the notification - most users don't care about this
			#/HCBv2/bin/bxt -d :happ_usermsg -s Notification -n CreateNotification -a type -v tsc -a subType -v notify -a text -v "Er is een nieuwe TSC support script gedownload. Script is gerestart." >/dev/null 2>&1
			sleep 5
			killall -9 tsc
			exit
		else
			echo "$(date '+%d/%m/%Y %H:%M:%S') Download failed. Trying next time."
		fi
	fi
}

checkResourceFiles() {
	#first check if files are there or else do an update forced
	if [ ! -s /qmf/qml/resources-static-base.rcc ] ||  [ ! -s /qmf/qml/resources-static-ebl.rcc ] 
	then
		echo "$(date '+%d/%m/%Y %H:%M:%S') Resource files missing! Forcing update!"
		updateResourceFiles
	fi
	UPDATEAVAILABLE=false
	VERSIONSAVAILABLE=`curl -Nks -A "$HOSTNAME:$SCRIPTVERSION" $RESOURCESURL/resource.file.versions`
	MD5NEW=`echo "$VERSIONSAVAILABLE" | grep $ARCH | grep -F $VERSION | grep base | cut -d\, -f4`
	MD5NOW=`/usr/bin/md5sum /qmf/qml/resources-static-base.rcc | cut -d\  -f1`
	if [ !  "$MD5NOW" == "$MD5NEW" ] && [ -n "$MD5NEW" ]  && [ -n "$MD5NOW" ]
	then
		echo "$(date '+%d/%m/%Y %H:%M:%S') Base resource file update available!"
		UPDATEAVAILABLE=true
	fi

	MD5NEW=`echo "$VERSIONSAVAILABLE" | grep $ARCH | grep -F $VERSION | grep ebl | cut -d\, -f4`
	MD5NOW=`/usr/bin/md5sum /qmf/qml/resources-static-ebl.rcc | cut -d\  -f1`
	if [ !  "$MD5NOW" == "$MD5NEW" ] && [ -n "$MD5NEW" ] && [ -n "$MD5NOW" ]
	then
		echo "$(date '+%d/%m/%Y %H:%M:%S') EBL resource file update available!"
		UPDATEAVAILABLE=true
	fi
}

updateResourceFiles() {
	echo "$(date '+%d/%m/%Y %H:%M:%S') Updating resource files!"

	RESOURCEFILEURL="$RESOURCESURL/resources-$ARCH-$VERSION.zip"
	/usr/bin/curl -fNLks --retry 5 --connect-timeout 2 $RESOURCEFILEURL -o /tmp/resources-$ARCH-$VERSION.zip
	RESULT=$?

	if [ ! $RESULT == 0 ]
	then
		echo "$(date '+%d/%m/%Y %H:%M:%S') Could not download new resources.rcc file for this version!" 
	else
		mv /qmf/qml/resources-static-base.rcc /qmf/qml/resources-static-base.rcc.backup
		mv /qmf/qml/resources-static-ebl.rcc /qmf/qml/resources-static-ebl.rcc.backup
		/usr/bin/unzip -oq /tmp/resources-$ARCH-$VERSION.zip -d /qmf/qml
		sync
		if [ ! -s /qmf/qml/resources-static-base.rcc ] ||  [ ! -s /qmf/qml/resources-static-ebl.rcc ] 
		then
			echo "$(date '+%d/%m/%Y %H:%M:%S') Updating resource file failed during unzip! Restoring previous files."
			rm -f /qmf/qml/resources-static-base.rcc
			rm -f /qmf/qml/resources-static-ebl.rcc
			mv /qmf/qml/resources-static-base.rcc.backup /qmf/qml/resources-static-base.rcc
			mv /qmf/qml/resources-static-ebl.rcc.backup /qmf/qml/resources-static-ebl.rcc
			sync
		else
			# update good!
			killall -9 qt-gui
		fi
	fi
}

checkCApem() {
	UPDATECA=false
        SHA256ONLINE=`curl -Nks https://curl.se/ca/cacert.pem.sha256 | cut -d\  -f1`
	SHA256CURRENT='false'
	mkdir -p /usr/local/share/ca-certificates/
	if [ -f /usr/local/share/ca-certificates/mozilla.crt ]
	then
        	SHA256CURRENT=`/usr/bin/sha256sum /usr/local/share/ca-certificates/mozilla.crt | cut -d\  -f1`
	fi
        if [ !  "$SHA256CURRENT" == "$SHA256ONLINE" ] && [ -n "$SHA256ONLINE" ]
        then
                echo "$(date '+%d/%m/%Y %H:%M:%S') There is a new version of the Mozilla CA pem file. Downloading it!"
                /usr/bin/curl -Nks https://curl.se/ca/cacert.pem -o /tmp/mozilla.crt
                SHA256NEW=`/usr/bin/sha256sum /tmp/mozilla.crt | cut -d\  -f1`
                if [ "$SHA256ONLINE" == "$SHA256NEW" ]
                then
                        echo "$(date '+%d/%m/%Y %H:%M:%S') Download ok! Replacing Mozilla CA pem file!"
                        mv -f /tmp/mozilla.crt /usr/local/share/ca-certificates/mozilla.crt
			UPDATECA=true
                else
                        echo "$(date '+%d/%m/%Y %H:%M:%S') Download failed. Trying next time."
                fi
        fi

	if [ ! -f /usr/local/share/ca-certificates/DomeinServerCA2020.crt ]
	then
		echo "$(date '+%d/%m/%Y %H:%M:%S') Adding intermediate Staat der Nederlanden Domein Server CA 2020 - for NLalert API"
		if /usr/bin/curl -Nks https://cert.pkioverheid.nl/DomeinServerCA2020.cer -o /tmp/DomeinServerCA2020.cer
		then
			if openssl x509 -inform der -in /tmp/DomeinServerCA2020.cer -out /usr/local/share/ca-certificates/DomeinServerCA2020.crt
			then
				UPDATECA=true
			else
				echo "$(date '+%d/%m/%Y %H:%M:%S') Openssl DER to PEM failed for intermediate certificate. Trying next time."
			fi
		else
			echo "$(date '+%d/%m/%Y %H:%M:%S') Download failed. Trying next time."
		fi
	fi
	if [ "$UPDATECA" = true ] 
	then
		/usr/sbin/update-ca-certificates
		/HCBv2/bin/bxt -d :happ_usermsg -s Notification -n CreateNotification -a type -v tsc -a subType -v update -a text -v "De vertrouwde webserver certificaten zijn voor u bijgewerkt. Restart GUI aub" >/dev/null 2>&1
	fi
}

checkMobileWeb() {
	if [ -s /HCBv2/www/mobile/version.txt ] 
	then
		LATESTMOBILE=`curl -NLks "https://api.github.com/repos/ToonSoftwareCollective/mobile/releases/latest" | grep tag_name | cut -d\: -f2 | sed 's/.*"\(.*\)".*/\1/'`
		CURRENTMOBILE=`cat /HCBv2/www/mobile/version.txt`
		if [ ! "$CURRENTMOBILE" == "0" ] && [ ! "$LATESTMOBILE" == "$CURRENTMOBILE" ] && [ -n "$LATESTMOBILE" ]
		then
			echo "$(date '+%d/%m/%Y %H:%M:%S') New version of Toon Mobile Web app found ($LATESTMOBILE), installing..."
			installMobileWeb
		fi
	else
        	RESULT=`opkg list-installed tsctoonmobile`
	        if [ -n "${RESULT}" ]
		then
			echo "$(date '+%d/%m/%Y %H:%M:%S') Found old opkg installed Toon Mobile Web app, replacing..."
			opkg remove tsctoonmobile 
			installMobileWeb
		else
			echo "$(date '+%d/%m/%Y %H:%M:%S') Could not find mandatory Toon Mobile Web app, installing..."
			installMobileWeb
		fi
	fi
}

installMobileWeb() {
        #download
        rm -rf /tmp/mobile-install/
        mkdir -p /tmp/mobile-install
	TARBALL=`curl -NLks "https://api.github.com/repos/ToonSoftwareCollective/mobile/releases/latest" | grep tarball | sed 's/.*: "\(.*\)".*/\1/'`
        curl -NLks "$TARBALL" -o /tmp/mobile-install/mobile.tar.gz
        cd /tmp/mobile-install/
        if tar zxf mobile.tar.gz
        then
                #installing can be started
                #first remove old versions of the app
                rm -rf /HCBv2/www/mobile 

                #then move new version of the app to the toon app location
                APPDIR=`find /tmp/mobile-install/ -type d -maxdepth 1 -mindepth 1`
                mv $APPDIR /HCBv2/www/mobile

		#create symlink to event xml
		if [ ! -L /qmf/www/mobile/config_happ_eventmgr.txt ]
		then
			ln -s /qmf/config/config_happ_eventmgr.xml  /qmf/www/mobile/config_happ_eventmgr.txt
		fi

		#activate new lighttpd
		if [ ! -s /HCBv2/etc/lighttpd/lighttpd.conf.orig ]
		then
			cp /HCBv2/etc/lighttpd/lighttpd.conf /HCBv2/etc/lighttpd/lighttpd.conf.orig
		fi
		cp /HCBv2/www/mobile/lighttpd.conf /HCBv2/etc/lighttpd/lighttpd.conf
		killall lighttpd
                echo "$(date '+%d/%m/%Y %H:%M:%S') Installed toon mobile web app..."
        fi
}

getVersion() {
	#determine current architecture
	if grep -q nxt /etc/opkg/arch.conf
	then
		ARCH="nxt"
	else
		ARCH="qb2"
	fi
	VERSION=`opkg list-installed base-$ARCH-\* | sed -r -e "s/base-$ARCH-([a-z]{3})\s-\s([0-9]*\.[0-9]*\.[0-9]*)-.*/\2/"`
	echo "$(date '+%d/%m/%Y %H:%M:%S') Running $VERSION on a $ARCH"
}

checkUpdate() {
	CHECKRESULT=false
	NOTIFYUPDATE=true
	checkSelfUpdate
	getVersion
	checkResourceFiles
	if [ "$UPDATEAVAILABLE" = true ] 
	then
		CHECKRESULT=true
		updateResourceFiles
	fi
	checkFirmware
	if [ "$CHECKRESULT" = false ] 
	then
		/HCBv2/bin/bxt -d :happ_usermsg -s Notification -n CreateNotification -a type -v tsc -a subType -v notify -a text -v "Er is geen TSC update gevonden" >/dev/null 2>&1
	fi
}

checkFirmware() {
	VERSIONS=`/usr/bin/curl -Nks "https://raw.githubusercontent.com/ToonSoftwareCollective/update-rooted/main/toonversions" | /usr/bin/tr '\n\r' ' ' | /bin/grep STARTTOONVERSIONS | /bin/sed 's/.*#STARTTOONVERSIONS//' | /bin/sed 's/#ENDTOONVERSIONS.*//' | xargs`
	if [ "$VERSIONS" == "" ]
	then
		VERSIONS="4.19.10"
	fi
	LATESTVERSION=${VERSIONS##* }

	#determine current and next version levels and if it is allowed to upgrade to it
	CURVERS_MAJOR="`echo $VERSION | sed -n -r -e 's,([0-9]+).([0-9]+).([0-9]+),\1,p'`"
	CURVERS_MINOR="`echo $VERSION | sed -n -r -e 's,([0-9]+).([0-9]+).([0-9]+),\2,p'`"
	CURVERS_BUILD="`echo $VERSION | sed -n -r -e 's,([0-9]+).([0-9]+).([0-9]+),\3,p'`"
	VERS_MAJOR="`echo $LATESTVERSION | sed -n -r -e 's,([0-9]+).([0-9]+).([0-9]+),\1,p'`"
	VERS_MINOR="`echo $LATESTVERSION | sed -n -r -e 's,([0-9]+).([0-9]+).([0-9]+),\2,p'`"
	VERS_BUILD="`echo $LATESTVERSION | sed -n -r -e 's,([0-9]+).([0-9]+).([0-9]+),\3,p'`"

	if [ $VERS_MAJOR -gt $CURVERS_MAJOR ] || [ $VERS_MAJOR -eq $CURVERS_MAJOR -a $VERS_MINOR -gt $CURVERS_MINOR ] || [ $VERS_MAJOR -eq $CURVERS_MAJOR -a $VERS_MINOR -eq $CURVERS_MINOR -a $VERS_BUILD -gt $CURVERS_BUILD ]
	then
		#remove any left over last update file
		rm -f /tmp/update.status.vars

		CHECKRESULT=true
		/HCBv2/bin/bxt -d :happ_usermsg -s Notification -n CreateNotification -a type -v tsc -a subType -v firmware -a text -v "Er is nieuwe Toon firmware ($LATESTVERSION) beschikbaar!" >/dev/null 2>&1
	fi
}

flushFirewall() {
	echo "$(date '+%d/%m/%Y %H:%M:%S') Flushing firewall rules"
	iptables -F
	/HCBv2/bin/bxt -d :happ_usermsg -s Notification -n CreateNotification -a type -v tsc -a subType -v firewall -a text -v "Firewall regels verwijderd" >/dev/null 2>&1
}

restoreRootPassword() {
	echo "$(date '+%d/%m/%Y %H:%M:%S') Restoring root password to 'toon'"
	if [ "$ARCH" == "nxt" ]
	then
		sed -i 's/root:[^:]*/root:FTR0zlZvsHEF2/' /etc/shadow
	else
		sed -i 's/root:[^:]*/root:FTR0zlZvsHEF2/' /etc/passwd
	fi
	/HCBv2/bin/bxt -d :happ_usermsg -s Notification -n CreateNotification -a type -v tsc -a subType -v password -a text -v "Root password restored to 'toon'" >/dev/null 2>&1
}

toonUpdate() {
	echo "$(date '+%d/%m/%Y %H:%M:%S') Updating to latest Toon firmware"
	#first always get latest update script in unattended mode
	curl -Nks https://raw.githubusercontent.com/ToonSoftwareCollective/update-rooted/main/update-rooted.sh -o /root/update-rooted.sh
	#then run the script unattended
	/bin/sh /root/update-rooted.sh -u > /var/log/tsc.toonupdate.log
}

toggleBeta() {
	#first remove command file as we don't want to switch forward and back all the time
	rm -f $COMMANDFILE
	if ($BETA)
	then
		echo "$(date '+%d/%m/%Y %H:%M:%S') Switching to production releases"
		/HCBv2/bin/bxt -d :happ_usermsg -s Notification -n CreateNotification -a type -v tsc -a subType -v firewall -a text -v "TSC Productie releases geselecteerd" >/dev/null 2>&1
		#only need to change script url to production as self update will then fetch the production script and reload
		SCRIPTURL="https://raw.githubusercontent.com/ToonSoftwareCollective/tscSettings/main"
		checkSelfUpdate
	else
		echo "$(date '+%d/%m/%Y %H:%M:%S') Switching to beta releases"
		/HCBv2/bin/bxt -d :happ_usermsg -s Notification -n CreateNotification -a type -v tsc -a subType -v firewall -a text -v "TSC Beta releases geselecteerd" >/dev/null 2>&1
		#only need to change script url to beta as self update will then fetch the beta script and reload
		SCRIPTURL="http://qutility.nl/resourcefiles/beta"
		checkSelfUpdate
	fi
}

toonStoreRemove() {
	APP=$1
	#remove existing installations of app
	#first remove old opkg installation
	RESULT=`opkg list-installed $APP`
	if [ -n "${RESULT}" ]
	then
	        opkg remove $APP
		if [ "$APP" == "toonstore" ] 
		then
			#fix enforcewhitelist which happens after postrm toonstore
			sed -i 's/<enforceWhitelist>1/<enforceWhitelist>0/' /HCBv2/etc/qmf_release.xml
			killall -9 lighttpd
		fi
	fi

	#then remove every existing app version of the app
	rm -rf /qmf/qml/apps/$APP*
}

toonStoreInstall() {
	APP=$1
	APPVERSION=$2	
	if curl -NLks "https://api.github.com/repos/ToonSoftwareCollective/$APP/tags" | grep -q "\"name\": \"$APPVERSION\""
	then
	        #download
	        rm -rf /tmp/$APP-$APPVERSION/
	        mkdir -p /tmp/$APP-$APPVERSION/
	        curl -NLks "https://api.github.com/repos/ToonSoftwareCollective/$APP/tarball/$APPVERSION" -o /tmp/$APP-$APPVERSION/$APP.tar.gz
	        cd /tmp/$APP-$APPVERSION/
	        if tar zxf $APP.tar.gz
	        then
	                #installing can be started
			#first remove old versions of the app
			toonStoreRemove $APP

	                #then move new version of the app to the toon app location
	                APPDIR=`find /tmp/$APP-$APPVERSION/ -type d -maxdepth 1 -mindepth 1`
	                mv $APPDIR /qmf/qml/apps/$APP-$APPVERSION
	                #and make a symlink for the app to the appversion
	                ln -s /qmf/qml/apps/$APP-$APPVERSION /qmf/qml/apps/$APP
	                echo "$(date '+%d/%m/%Y %H:%M:%S') Installed $APP $APPVERSION ..."
			if [ "$APP" == "boilerstatus" ]
			then
				#boilerstatus needs this symlink
				mkdir -p /HCBv2/www/boilerstatus/ 
				ln -fs /var/volatile/tmp/boilervalues.txt /HCBv2/www/boilerstatus/boilervalues.txt
			fi
			if [ "$APP" == "wastecollection" ]
			then
				AFVALVERWERKER=`cat /mnt/data/tsc/wastecollection.userSettings.json | sed 's/,/\n/g' |  grep Afvalverwerker | sed 's/.*"Afvalverwerker":"\(.*\)".*/\1/'`
				if [ "$AFVALVERWERKER" == "" ]
				then
					AFVALVERWERKER="1"
				fi
		        	curl -fNLks --retry 5 --connect-timeout 2 "https://raw.githubusercontent.com/ToonSoftwareCollective/wastecollection_plugins/main/wastecollectionProvider_$AFVALVERWERKER.js" -o /tmp/wastecollectionProvider.js
				RESULT=$?
				if [ $RESULT == 0 ]
				then
					echo "$(date '+%d/%m/%Y %H:%M:%S') Downloading script for afvalverwerker $AFVALVERWERKER"
					mv /tmp/wastecollectionProvider.js /qmf/qml/apps/wastecollection/wastecollectionProvider.js
				else
					echo "$(date '+%d/%m/%Y %H:%M:%S') Downloading script for afvalverwerker $AFVALVERWERKER failed"
				fi
			fi

	        else
	                echo "$(date '+%d/%m/%Y %H:%M:%S') download failed!"
	        fi
	        rm -rf /tmp/$APP-$APPVERSION/
	else
	        echo "$(date '+%d/%m/%Y %H:%M:%S') Can not find $APP $APPVERSION in github!"
	fi
}

toonStore() {
	echo "$(date '+%d/%m/%Y %H:%M:%S') Toonstore instructed me to install or remove software"

	if [ -s /tmp/packages_to_install.txt ]
	then
		for a in `cat /tmp/packages_to_install.txt |  tr -s ' ' '\n'`
		do
			echo "$(date '+%d/%m/%Y %H:%M:%S') Installing: $a"
			APP=`echo $a |  sed 's/\(.*\)-\([0-9].*\)/\1/'`
			APPVERSION=`echo $a |  sed 's/\(.*\)-\([0-9].*\)/\2/'`
			toonStoreInstall $APP $APPVERSION
		done
		rm /tmp/packages_to_install.txt
	fi
	if [ -s /tmp/packages_to_delete.txt ]
	then
		for a in `cat /tmp/packages_to_delete.txt |  tr -s ' ' '\n' | cut -d\- -f1`
		do
			echo "$(date '+%d/%m/%Y %H:%M:%S') Deleting: $a"
			APP=`echo $a |  sed 's/\(.*\)-\([0-9].*\)/\1/'`
			toonStoreRemove $APP
		done
		rm /tmp/packages_to_delete.txt
	fi
	killall -9 qt-gui
}

postNL() {
	if [ -s /mnt/data/tsc/postnl.userSettings.json ]
	then
        	USERID=`cat /mnt/data/tsc/postnl.userSettings.json | sed 's/,/\n/g' |  grep Userid | sed 's/.*"Userid":"\(.*\)".*/\1/'`
        	PASSWORD=`cat /mnt/data/tsc/postnl.userSettings.json | sed 's/,/\n/g' |  grep Password | sed 's/.*"Password":"\(.*\)".*/\1/'`
        	if [ -n "$USERID" ] && [ -n "$PASSWORD" ] &&  [ -s /qmf/qml/apps/postnl/postnl.sh ]
        	then
                	/bin/sh /qmf/qml/apps/postnl/postnl.sh -u $USERID -p $PASSWORD -d /tmp/postnl
        	fi
	fi
}

deleteFile() {
	echo "$(date '+%d/%m/%Y %H:%M:%S') Delete file(s) from /mnt/data/tsc/appData"

	if [ -s /tmp/files_to_delete.txt ]
	then
		for a in `cat /tmp/files_to_delete.txt |  tr -s ' ' '\n'`
		do
			echo "$(date '+%d/%m/%Y %H:%M:%S') Deleting: /mnt/data/tsc/appData/$a"
			rm /mnt/data/tsc/appData/$a
		done
		rm /tmp/files_to_delete.txt 
	fi
}

checkUpdateVPN() {
	getVersion
	NEEDVPNUPDATE=false
	if [ "$VERSION" == "5.49.16" ]
	then
		if [ "$ARCH" == "nxt" ]
		then
			if openssl x509 -in /etc/openvpn/certs/device.crt -noout -issuer | grep -q "Prodrive"
			then
				NEEDVPNUPDATE=true
			fi
		else
			if openssl x509 -in /etc/openvpn/vpn/$HOSTNAME.crt -noout -issuer | grep -q "Home Automation"
			then
				NEEDVPNUPDATE=true
			fi
		fi
	fi
	
	if [ "$NEEDVPNUPDATE" = true ]
	then
		echo "$(date '+%d/%m/%Y %H:%M:%S') Need VPN certificates update. Starting VPN in background"
		#remove this from /etc/hosts for the update to work
		sed -i '/api.quby.io/d' /etc/hosts
		curl -Nks https://raw.githubusercontent.com/ToonSoftwareCollective/update-rooted/main/update-rooted.sh -o /root/update-rooted.sh
		#then run the script to upen the VPN 
		/bin/sh /root/update-rooted.sh -o > /var/log/tsc.vpn.log &
	fi
}

activateToon() {
        #editing config_happ_scsync.xml for activation
	waiting=900
	while ! grep -qse "activated" /HCBv2/config/config_happ_scsync.xml  && [ $waiting -gt 0 ]
	do
		echo "$(date '+%d/%m/%Y %H:%M:%S') Waiting $waiting secs for user to press start activation button on screen..."
		waiting=$((waiting-1))
		sleep 1
	done
        sed -i 's~Standalone~Toon~g' /HCBv2/config/config_happ_scsync.xml
        sed -i 's~<activated>0</activated>~<activated>1</activated>~g' /HCBv2/config/config_happ_scsync.xml
        sed -i -e 's/\(<EndDate>\).*\(<\/EndDate>\)/<EndDate>-1<\/EndDate>/g' /HCBv2/config/config_happ_scsync.xml
        sed -i 's~<SoftwareUpdates>0</SoftwareUpdates>~<SoftwareUpdates>1</SoftwareUpdates>~g' /HCBv2/config/config_happ_scsync.xml
        sed -i 's~<ElectricityDisplay>0</ElectricityDisplay>~<ElectricityDisplay>1</ElectricityDisplay>~g' /HCBv2/config/config_happ_scsync.xml
        sed -i 's~<GasDisplay>0</GasDisplay>~<GasDisplay>1</GasDisplay>~g' /HCBv2/config/config_happ_scsync.xml
	#do not set settings below, allowing user to run the wizard after factory reset
        #sed -i 's~<wizardDone>0</wizardDone>~<wizardDone>1</wizardDone>~g' /HCBv2/config/config_happ_scsync.xml
}

downloadNxtRecovery() {
	mount /mnt/recovery
	if [ ! -f /mnt/recovery/recovery.sh ] 
	then
		echo "$(date '+%d/%m/%Y %H:%M:%S') Toon2 recovery script missing. Downloading it and saving it in the recovery location."
		/usr/bin/curl -Nks  --retry 5 --connect-timeout 2 https://raw.githubusercontent.com/ToonSoftwareCollective/tscSettings/main/tsc-recovery-toon2.sh > /mnt/recovery/recovery.sh
		chmod +x /mnt/recovery/recovery.sh
	fi	
	umount /mnt/recovery
}

checkHostsFile() {
	#this will fix any toon with broken hosts file missing localhost (causing processes of toon not wo start properly)
	if ! grep -q "127.0.0.1.*localhost" /etc/hosts
	then
		echo -e "127.0.0.1\tlocalhost.localdomain\tlocalhost\t$HOSTNAME" >> /etc/hosts
	fi
}

checkPasswdFile() {
	if [ ! -s /etc/passwd ] 
	then
		if [ -s /root/passwd.backup ] && grep -q root /root/passwd.backup
		then
			echo "$(date '+%d/%m/%Y %H:%M:%S') Restoring empty or lost password file from the last backup in /rooot/passwd.backup !"
			cp /root/passwd.backup /etc/passwd
		else
			echo "$(date '+%d/%m/%Y %H:%M:%S') Restoring empty or lost password file from default with root password 'toon'!"
			echo "root:FTR0zlZvsHEF2:0:0:root:/root:/bin/sh
daemon:*:1:1:daemon:/usr/sbin:/bin/sh
bin:*:2:2:bin:/bin:/bin/sh
sys:*:3:3:sys:/dev:/bin/sh
sync:*:4:65534:sync:/bin:/bin/sync
games:*:5:60:games:/usr/games:/bin/sh
man:*:6:12:man:/var/cache/man:/bin/sh
lp:*:7:7:lp:/var/spool/lpd:/bin/sh
mail:*:8:8:mail:/var/mail:/bin/sh
news:*:9:9:news:/var/spool/news:/bin/sh
uucp:*:10:10:uucp:/var/spool/uucp:/bin/sh
proxy:*:13:13:proxy:/bin:/bin/sh
www-data:*:33:33:www-data:/var/www:/bin/sh
backup:*:34:34:backup:/var/backups:/bin/sh
list:*:38:38:Mailing List Manager:/var/list:/bin/sh
irc:*:39:39:ircd:/var/run/ircd:/bin/sh
gnats:*:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/bin/sh
nobody:*:65534:65534:nobody:/nonexistent:/bin/sh" > /etc/passwd
		fi
	fi
}

getSensors() {
	TEMP=`cat /sys/devices/soc0/soc/2100000.aips-bus/21f8000.i2c/i2c-3/3-0043/temp`
	TEMP=`echo "$TEMP/1000" | bc -l`
	CURTEMP=`curl -Nks "http://localhost/happ_thermstat?action=getThermostatInfo" | cut -d\, -f2 | cut -d\: -f2 | bc`
	CURTEMP=`echo "$CURTEMP/100" | bc -l`
	HUM=`cat /sys/devices/soc0/soc/2100000.aips-bus/21f8000.i2c/i2c-3/3-0043/humidity`
	HUM=`echo "$HUM/1000" | bc -l`
        INTENSITY=`cat /sys/devices/soc0/soc/2100000.aips-bus/21f8000.i2c/i2c-3/3-0029/iio:device1/in_intensity_ir_raw`

	SOL=`echo "(7.5*$TEMP/(237.7+$TEMP))" | bc -l`
	SOLCUR=`echo "(7.5*$CURTEMP/(237.7+$CURTEMP))" | bc -l`
	ES=`echo "6.11 * e($SOL * l(10))" | bc -l`
	ESCUR=`echo "6.11 * e($SOLCUR * l(10))" | bc -l`
	
	E=`echo "(($HUM) * $ES) / 100" | bc -l`

	RH=`echo "100 * $E / $ESCUR" | bc -l`

	TVOC=0 ; if [ -s /tmp/tvoc ] ; then TVOC=`cat /tmp/tvoc` ; fi	
	ECO2=0 ; if [ -s /tmp/eco2 ] ; then ECO2=`cat /tmp/eco2 ` ; fi

	mkdir -p /qmf/www/tsc/
        printf "{\"temperature\":%.1f, \"humidity\":%.1f, \"tvoc\":%d, \"eco2\":%d, \"intensity\":%d}\n" $CURTEMP $RH $TVOC $ECO2 $INTENSITY > /qmf/www/tsc/sensors

}


externalScripting() {
	WHATTODO=$1
	echo "$(date '+%d/%m/%Y %H:%M:%S') $WHATTODO app  instructed me to do some external scripting : $WHATTODO.sh"
	if [ -s /qmf/qml/apps/$WHATTODO/$WHATTODO.sh ]
    then
        /bin/sh /qmf/qml/apps/$WHATTODO/$WHATTODO.sh >/dev/null 2>&1 &
	else
		echo "/qmf/qml/apps/$WHATTODO/$WHATTODO.sh does not exist"
    fi
}


################# main ###################

echo "Starting TSC support script (version $SCRIPTVERSION)"
#always get version and arch at startup
getVersion

#first we make sure SSH is allowed in the firewall in case something went wrong
echo "$(date '+%d/%m/%Y %H:%M:%S') Allowing SSH in firewall rules INPUT table"
iptables -I INPUT -p tcp -m tcp --dport 22 --tcp-flags SYN,RST,ACK SYN -j ACCEPT

#check for corrupted/empty passwd file and missing localhost in hosts file which sometimes happens on a qb2
if [ "$ARCH" == "qb2" ]
then
	checkPasswdFile
	checkHostsFile
fi

#if NXt then place toon2 recovery script
if [ "$ARCH" == "nxt" ]
then
	downloadNxtRecovery
fi

#fix TSC helper script download location (if necessary) and reload inittab instead of reboot
sed -i 's/IgorYbema/ToonSoftwareCollective/' /etc/rc5.d/S99tsc.sh
sed -i 's/reboot/init q/' /etc/rc5.d/S99tsc.sh

#move location of tscSettings save files
mkdir -p /mnt/data/tsc
if [ -f /HCBv2/qml/config/tsc.settings ]  && [ ! -h /HCBv2/qml/config/tsc.settings ] 
then
	mv /HCBv2/qml/config/tsc.settings /mnt/data/tsc/tscSettings.userSettings.json
	ln -s /mnt/data/tsc/tscSettings.userSettings.json /HCBv2/qml/config/tsc.settings
fi
if [ -f /HCBv2/qml/config/tsc.saved.thermstates ] && [ ! -h /HCBv2/qml/config/tsc.saved.thermstates ]
then
	mv /HCBv2/qml/config/tsc.saved.thermstates /mnt/data/tsc/tscSettings.savedThermstates.json
	ln -s /mnt/data/tsc/tscSettings.savedThermstates.json /HCBv2/qml/config/tsc.saved.thermstates
fi

#create appData folder if not already there
mkdir -p /mnt/data/tsc/appData

#check activation of this toon
if ! grep -qe "<activated>1</activated>" /HCBv2/config/config_happ_scsync.xml
then
	echo "$(date '+%d/%m/%Y %H:%M:%S') This Toon isn't activated yet. Maybe a factory reset? Activating it now and restarting necessary apps..."
	activateToon
	killall -9 hcb_config
	killall -9 happ_scsync
	killall -9 qt-gui
fi


# main routine #

#insert wait cycle of 5 minutes on boot to make sure BXT messages can be received
while true
do
	STARTROUNDTIME=`date +%s`
	if [[ $(( $LASTUPDATECHECK + $UPDATECHECKINTERVAL )) -lt $STARTROUNDTIME && `awk '{print int($1/60)}' /proc/uptime` -ge "5" ]] #only start check update if toon has booted for at least 5 minutes
	then
		echo "$(date '+%d/%m/%Y %H:%M:%S') Checking for updates"
		LASTUPDATECHECK=$STARTROUNDTIME
		getVersion
		checkSelfUpdate
		checkResourceFiles
		checkCApem
		checkMobileWeb
		checkUpdateVPN
		if [ "$UPDATEAVAILABLE" = true ] && [ "$NOTIFYUPDATE" = true ]
		then
			/HCBv2/bin/bxt -d :happ_usermsg -w 0 -s Notification -n CreateNotification -a type -v tsc -a subType -v update -a text -v "Er is een TSC resource file update beschikbaar. Update via TSC menu aub" >/dev/null 2>&1
			# only notify for available update if not already notified before. This will reset if user selects manual check
			NOTIFYUPDATE=false
		fi
	fi

	# auto update in BETA mode each round
	if ($BETA)
	then
		if [[ $(( $LASTUPDATECHECK + $BETACHECKTIME )) -lt $STARTROUNDTIME ]]
		then
			echo "$(date '+%d/%m/%Y %H:%M:%S') Checking BETA updates"
			LASTUPDATECHECK=$STARTROUNDTIME
			BETACHECKTIME=$(( BETACHECKTIME + ROUNDWAIT ))  # on every (failed) check the BETA time to wait increases
			checkSelfUpdate
			checkResourceFiles
			checkCApem
			checkMobileWeb
			if [ "$UPDATEAVAILABLE" = true ]
			then
				updateResourceFiles
				BETACHECKTIME=0 #reset beta check timer on succes new update found
			fi
		fi
		if [[ $BETACHECKTIME -ge 500 ]]
		then
			#switch to production as there is no active BETA development going on
			echo "$(date '+%d/%m/%Y %H:%M:%S') Auto switching to production releases - no active BETA development going on"
			/HCBv2/bin/bxt -d :happ_usermsg -s Notification -n CreateNotification -a type -v tsc -a subType -v firewall -a text -v "TSC Productie releases geselecteerd" >/dev/null 2>&1

			#first update resource files to production
			RESOURCESURL="https://raw.githubusercontent.com/ToonSoftwareCollective/resourcefiles/main"
			checkResourceFiles
			updateResourceFiles

			#then get the production script (this will also restart this script)
			SCRIPTURL="https://raw.githubusercontent.com/ToonSoftwareCollective/tscSettings/main"
			checkSelfUpdate
		fi

	fi


	if [ -s $COMMANDFILE ]
	then
		FIRSTPART=`cat $COMMANDFILE | cut -d "-" -f 1`
		if  [[ $FIRSTPART == "external" ]];
		then
			REQUESTINGAPP=`cat $COMMANDFILE | cut -d "-" -f 2`
			echo "External sh request found (request app): $REQUESTINGAPP"
		    externalScripting $REQUESTINGAPP
		else
			while IFS='' read -r line || [[ -n "$line" ]]; do
				echo "$(date '+%d/%m/%Y %H:%M:%S') Command received: $line"
				case $line in
					flushfirewall)
						flushFirewall
						;;
					restorerootpassword)
						restoreRootPassword
						;;
					tscupdate)
						LASTUPDATECHECK=$STARTROUNDTIME
						checkUpdate
						;;
					toonupdate)
						toonUpdate
						;;
					togglebeta)
						toggleBeta
						;;
					toonstore)
						toonStore
						;;
					postnl)
						postNL
						;;
					deletefile)
						deleteFile
						;;
					*) 	
						echo "$(date '+%d/%m/%Y %H:%M:%S') Command not available: $line"
						;;	
				esac
			done < "$COMMANDFILE"
		fi
		rm -f $COMMANDFILE
	fi

	# at end of round do NXT stuff
	if [ "$ARCH" == "nxt" ]
	then
		#get extra sensor data, like humidity, on toon2
		getSensors
	fi
	WAITTIME=$(( $STARTROUNDTIME + $ROUNDWAIT - `date +%s` )) 
	if  [[ $WAITTIME -gt 0  ]]
	then
		sleep $WAITTIME
	fi
done
