# wpa_supplicant for UDM and UDM Pro
## UPDATED 04.08.2022

## overview
This guide has primarily been written for authenticating to AT&T U-Verse using wpa_supplicant on a UDM Pro.  It is assumed that you've already retrieved your certificates from a modem supplied by AT&T.  If you have not, you can purchase a used modem on ebay, such as the NVG589 and then root it to get the certificates.  I had success using the following guide.


https://github.com/bypassrg/att


If the above link no longer works, I have also forked it to my GitHub below.

https://github.com/pbrah/att

Before moving on to the next section, make sure you've copied your root certificate into the CA_*.pem file per the instructions of the 802.1x Credential Extraction Tool (mfg_dat_decode).

## running the Podman image
1.  scp your certs and wpa_supplicant.conf to the UDM Pro

```
scp -r *.pem root@192.168.1.1:/tmp/
root@192.168.1.1's password:
CA_001E46-xxxx.pem                                                          100% 3926     3.8KB/s   00:00
Client_001E46-xxxx.pem                                                      100% 1119     1.1KB/s   00:00
PrivateKey_PKCS1_001E46-xxxx.pem                                            100%  887     0.9KB/s   00:00

scp -r wpa_supplicant.conf root@192.168.1.1:/tmp/
wpa_supplicant.conf                                                         100%  680     0.7KB/s   00:00
```

2. ssh to the UDM Pro, create a directory for the certs and wpa_supplicant.conf in the docker directory then copy the files over.

```
mkdir /mnt/data/podman/wpa_supplicant/
cp -arfv /tmp/*pem /tmp/wpa_supplicant.conf /mnt/data/podman/wpa_supplicant/
```

3. Update the wpa_supplicant.conf to reflect the correct paths for our container.  **Do not run these more than once or you will end up with incorrect paths.**

```
sed -i 's,ca_cert=",ca_cert="/etc/wpa_supplicant/conf/,g' /mnt/data/podman/wpa_supplicant/wpa_supplicant.conf
sed -i 's,client_cert=",client_cert="/etc/wpa_supplicant/conf/,g' /mnt/data/podman/wpa_supplicant/wpa_supplicant.conf
sed -i 's,private_key=",private_key="/etc/wpa_supplicant/conf/,g' /mnt/data/podman/wpa_supplicant/wpa_supplicant.conf
```

4. Pull the docker image while you have an internet connection on the UDM Pro. 

5. Run the wpa_supplicant docker container, the docker run command below assumes you are using port 9 or eth8 for your wan.  If not, adjust accordingly.

```
podman run --privileged --network=host --name=wpa_supplicant-udmpro -v /mnt/data/podman/wpa_supplicant/:/etc/wpa_supplicant/conf/ --log-driver=k8s-file --restart always -d -ti pbrah/wpa_supplicant-udmpro:v1.0 -Dwired -ieth8 -c/etc/wpa_supplicant/conf/wpa_supplicant.conf
```
6. Install onboot.d (scripts in the /mnt/data/onboot.d directory will run on boot) 

```
unifi-os shell
curl -L https://udm-boot.boostchicken.dev -o udm-boot_1.0.5_all.deb
dpkg -i udm-boot_1.0.5_all.deb
exit
```
7.  Next, lets add the startup script: 

```
vi /mnt/data/on_boot.d/10-wpa_supplicant.sh
```

#!/bin/sh

podman start wpa_supplicant-udmpro


8. Update the permissions on the script:
```
chmod +x /mnt/data/on_boot.d/10-wpa_supplicant.sh

```

Before rebooting, make sure to update the mac address in the Network>Settings>Internet for the WAN port or it will not work.


## troubleshooting
If you are having issues connecting after starting your docker container, the first thing you should do is check your docker container logs.
```
podman logs -f wpa_supplicant-udmpro
```

From a recent case I assisted in troubleshooting, the user saw the following in their logs.  The was due to their wpa_supplicant.conf having incorrect paths to the certificates.  Refer to my example in the instructions to ensure yours are pointing to the correct location.
```
OpenSSL: tls_connection_ca_cert - Failed to load root certificates error:02001002:system library:fopen:No such file or directory
OpenSSL: pending error: error:2006D080:BIO routines:BIO_new_file:no such file
OpenSSL: pending error: error:0B084002:x509 certificate routines:X509_load_cert_crl_file:system lib
OpenSSL: tls_load_ca_der - Failed load CA in DER format error:02001002:system library:fopen:No such file or directory
OpenSSL: pending error: error:20074002:BIO routines:file_ctrl:system lib
OpenSSL: pending error: error:0B06F002:x509 certificate routines:X509_load_cert_file:system lib
TLS: Failed to set TLS connection parameters
EAP-TLS: Failed to initialize SSL.
```


### UDM non-pro
This works with the standard UDM as of OS 1.7.0 with a few adjustments to the podman run command:

```
podman run --privileged --network=host --name=wpa_supplicant-udmpro -v /mnt/data/podman/wpa_supplicant/:/etc/wpa_supplicant/conf/ --log-driver=k8s-file --restart always -d -ti pbrah/wpa_supplicant-udmpro:v1.0 -Dwired -ieth4 -c/etc/wpa_supplicant/conf/wpa_supplicant.conf
```


https://wells.ee/journal/2020-08-05-bypassing-att-fiber-modem-udmp/
