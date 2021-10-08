# asus-router-decoder
Decode text from ASUS wireless router obfuscated saved config files

This bash script can be used to recover plaintext SSID, WPA PSK, and HTTP admin password from an ASUS RT-AC3100 Wireless Router.

The config file can be generated from the `Administration - Restore/Save/Upload Setting` page and will be named something like `Settings_RT-AC3100.CFG`.

```bash
#!/usr/bin/bash
# adapted from source:
# https://github.com/RMerl/asuswrt-merlin.ng/blob/master/release/src/router/nvram/nvram.c#L777

if [ "$#" -ne 1 ]
then
   echo "Supply the ASUS router config file name."
   exit
fi

file=$1
size=$(stat -c %s "$file")

if [ "$size" -lt 10 ]
then
   echo "File size too small."
   exit
fi

fdata=( $(cat "$file" | xxd -p | fold -w 2) )

if [ "${#fdata[@]}" -ne "$size" ]
then
   echo "File read error."
   exit
fi

if [ "${fdata[0]}" != "48" ] || [ "${fdata[1]}" != "44" ] || [ "${fdata[2]}" != "52" ] || [ "${fdata[3]}" != "32" ]
then
   echo "File header check failed."
   exit
fi

dlen=""
for i in {6..4}
do
   dlen=$dlen${fdata[$i]}
done
dlen=$((0x$dlen))

if [ "$dlen" -ne "$(($size-8))" ]
then
   echo "Data length check failed."
   exit
fi

rand=${fdata[7]}
i="8"
out=$(mktemp)

echo "[+] Decoding config file..."
while [ "$i" -lt "$size" ]
do
   if [ "$((0x${fdata[$i]}))" -gt 252 ]
   then
      if [ "$i" -gt 8 ] && [ "${fdata[$(($i-1))]}" != "00" ]
      then
         echo -n -e "\x00" >> $out
      else
         :
      fi
   else
      b=$(printf "%02x" $((0xff+0x$rand-0x${fdata[$i]})))
      echo -n -e "\x$b" >> $out
   fi
   i=$(($i+1))
done

echo -e "[+] Attempting to recover:\n    SSID, WPA PSK, HTTP admin password"
strings $out | grep "_wpa_psk=\|_ssid=\|http_passwd="
rm $out
```
