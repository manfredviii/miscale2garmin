# Mi Body Composition Scale 2 Garmin Connect

## 1. Introduction
- This project is based on the following projects:
  - https://github.com/RobertWojtowicz/miscale2garmin;
  - https://github.com/davidkroell/bodycomposition;
  - https://github.com/wiecosystem/Bluetooth;
  - https://github.com/lolouk44/xiaomi_mi_scale;
- It allows the Mi Body Composition Scale 2 (model: XMTZC05HM) to be fully automatically synchronized to Garmin Connect, the following parameters:
  - BMI;
  - Body Fat;
  - Body Water;
  - Bone Mass;
  - Metabolic Age;
  - Physique Rating;
  - Skeletal Muscle Mass;
  - Time;
  - Visceral Fat;
  - Weight **_(kg units only)_**;
- Synchronization diagram from Mi Body Composition Scale 2 to Garmin Connect:

![alt text](https://github.com/manfredviii/miscale2garmin/blob/master/pic/workflow.png)

## 2. Getting the MAC address of Mi Body Composition Scale 2
- Install Zepp Life App on your mobile device, user manual: https://files.xiaomi-mi.com/files/smart_scales/smart_scales-EN.pdf;
- Configure your scale with Zepp Life App on your mobile device (tested on Android 10, 11, 12);
- Retrieve scale's MAC Address from Zepp Life App (Profile > My devices > Mi Body Composition Scale 2):

![alt text](https://github.com/manfredviii/miscale2garmin//blob/master/pic/mac_addr.png)

## 3. Setting correct date and time in Mi Body Composition Scale 2
###    NB: YOU SHOULD ALSO SYNCRONIZE SCALE AFTER REPLACING BATTERIES
- Launch Zepp Life App, go to scale (Profile > My devices > Mi Body Composition Scale 2);
- Start scale and select Clear data in App;
- Take a new weight measurement with App, App should synchronize date and time;
- 2 times a year it is necessary to perform synchronization due to change from winter to summer time and then vice versa;
- NB: YOU SHOULD ALSO SYNCRONIZE SCALE AFTER REPLACING BATTERIES
- Script import_ble.sh and import_mqtt.sh have time difference detection of more than 20 minutes (between scale data and os);
- If time is still not synchronizing correctly, check NTP synchronization on your server.

## 4. RASPBERRY BLE VERSION
### 4.1. How does this work ?
- After weighing, Mi Body Composition Scale 2 is active for 15 minutes on bluetooth transmission;
- Raspberry Pi internal (or USB bluetooth device, tested with USB bluetooth versions 4.0 and 5.0) scans for BLE device every 1 minute for 10 seconds and queries scale for data;
- Body weight and impedance data on the server are appropriately processed by scripts;
- Processed data are sent by the program bodycomposition to Garmin Connect;
- Raw data from the scale is backed up on the server in backup.csv file;
- backup.csv file can be imported e.g. for analysis into Excel.

### 4.2. Preparing Linux system
- I tested on a Raspberry Pi 3 (model B)
- If you want a virtual machine with Debian (tested on 10 and 11), minimum hardware requirements are: 1 CPU, 512 MB RAM, 2 GB HDD, network connection;
- Update your system and then install following modules:
```
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install wget python3 bluetooth python3-pip libglib2.0-dev -y
sudo pip install bluepy
```
- Modify file
```sudo nano /etc/systemd/system/bluetooth.target.wants/bluetooth.service```:
```
ExecStart=/usr/lib/bluetooth/bluetoothd --noplugin=sap
```
- Download and extract to your home directory (e.g. "/home/pi/"), make a files executable (**_NOTE_: choose correct version of boodycomposition depending on your computer**, e.g. "_Linux_armv6.tar.gz" for Raspberry Pi3 or "_Linux_x86_64.tar.gz" for other computer):
```
wget https://github.com/RobertWojtowicz/miscale2garmin/archive/refs/tags/3.tar.gz -O - | tar -xz
cd miscale2garmin-3
wget https://github.com/davidkroell/bodycomposition/releases/download/v1.7.0/bodycomposition_1.7.0_Linux_armv6.tar.gz -O - | tar -xz bodycomposition
sudo chmod +x bodycomposition import_ble.sh scanner_ble.py export_garmin.py /usr/local/lib/python3.9/dist-packages/bluepy/btle.py
```

### 4.3. Configuring scripts
- First script is "scanner_ble.py", you need to complete data: "scale_mac_addr", which is related to the MAC address of the scale (es "0c:95:41:c8:f8:94");
- Script "scanner_ble.py" has implemented debug mode, you can verify if everything is working properly, just execute it from console:
```
$ sudo python3 /home/pi/miscale2garmin-3/scanner_ble.py
Mi Body Composition Scale 2 Garmin Connect v3.0 (scanner_ble.py)

* Starting BLE scan:
   BLE device found with address: 00:00:00:00:00:00, non-target device
   BLE device found with address: 00:00:00:00:00:00, non-target device
   BLE device found with address: 00:00:00:00:00:00, non-target device
   BLE device found with address: 0c:95:41:c8:f8:94 <= target device
   BLE device found with address: 0c:95:41:c8:f8:94 <= target device
   BLE device found with address: 0c:95:41:c8:f8:94 <= target device
* Reading BLE data complete, finished BLE scan
76.35;467;1646238431;2022-3-2 17:27:11
```
- Second script is "export_garmin.py", you must complete data in the "users" section: sex, height in cm, birthdate in dd-mm-yyyy, email and password to Garmin Connect, max_weight in kg, min_weight in kg;
- Script "export_garmin.py" supports multiple users with individual weights ranges, we can link multiple accounts with Garmin Connect
es:
```
users = [User("male", 172, '13-12-1983', "paolomonfredini@gmail.com", "Vivafm40", 78, 67), User("female", 164, '27-10-1981', "casa.monfredini@gmail.com", "Condino1", 66, 52)];
```
- Script "import_ble.sh" has implemented debug mode, you can verify if everything is working properly, just execute it from console:
```
$ sudo /home/pi/miscale2garmin-3/import_ble.sh
Mi Body Composition Scale 2 Garmin Connect v3.0 (import_ble.sh)

* Data backup file exists, checking for new data
* Importing and calculating data to upload
* Data upload to Garmin Connect is complete
```
- If there is an error upload to Garmin Connect, data will be sent again on the next execution, upload errors and other operations are saved in temp.log file:
```
$ cat /home/pi/miscale2garmin-3/temp.log
... uploading weight
Mi Body Composition Scale 2 Garmin Connect v3.0 (export_garmin.py)
Processed file: 1641199035.tlog
```
- Finally, if everything works correctly add script import_ble.sh to CRON to run it every 1 minute ```sudo crontab -e```:
```
*/1 * * * * /home/pi/miscale2garmin-3/import_ble.sh
```
