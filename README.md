# CanBus_Troubleshooting
Basic Troubleshooting for 3d Printer Canbus networks

Canbus networks are awesome recent addtions to modern 3D printers, but they're are extrmely complex & finicky things to setup & maintain. They're even more difficult to fault find without advanced knowledge!

Here's a few basic but effective steps & tips to diagnosing & fixing issues with your Canbus network.

************************************************
# To set up your Canbus network follow the Katapult documentation.
https://github.com/Arksine/katapult

If however you don't use Katapult & want to flash in DFU mode with no bootloader you can do that too. I might help remove a layer of complexity if you're having issues. 

THIS WILL OVERWRITE ANY KATAPULT BOOTLOADERS!
TO FLASH KATAPULT BACK AFTERWARDS YOU WILL NEED TO MANUALLY PUT THE NODE INTO DFU MODE AGAIN!

DO NOT DO THIS IF YOU THINK YOU WILL NOT BE ABLE TO GET KATAPULT BACK - Jump down to the next section!


Be sure you have created this file on your Pi
```
 sudo nano /etc/network/interfaces.d/can0
```
It should contain these lines...
```
 allow-hotplug can0
 iface can0 can static
   bitrate 1000000
   up ifconfig $IFACE txqueuelen 1024
```

Exit & Save

Reboot the pi if you made any changes/corrections.



Connect your Canbus node in DFU mode - see node's manual for this step!

log in to your SSH terminal....

```
lsusb
```
your SSH terminal should report something like in this example: 

`Bus 001 Device 006: ID 0483:df11 STMicroelectronics STM Device in DFU Mode`

Now type...
```
cd klipper
```
```
make menuconfig
```
Set firmware values for your selected node hardware. Paying special attention to set the correct Canbus frequency, the same as in the can0 file above (1000000)

Save & Exit

Now create the firmware & directly flash via DFU mode.
```
make flash FLASH_DEVICE=0483:df11
```
*change Serial ID for correct one for your node!

###### NOTE: Its safe to ignore any `failed to flash` warnings from the dfu-util as long as the terminal says `File downloaded successfully` above it.

Now safely disconnect the newly flashed node from the USB cable & remove any power over USB jumpers!!

************************************************

# Check Canbus network for unassigned UUID’s

Have only 1 unassigned node connected at once or it could cause a “Bus-off” state & the nodes will need to be reset (power cycled)
```
~/klippy-env/bin/python ~/klipper/scripts/canbus_query.py can0
```

If any unassigned UUID's are found the terminal will report:

`Found canbus_uuid=3754cb7564f3, Application: Klipper`

Copy the UUID to MCU section into the selected .cfg file for the printer to use.



## Canbus TIP:

This command can also be used as a diagnostic tool when the printer is showing a `unable to connect` error if you already have the node's UUID in the system but its not working & you get the red error screens.

Comment out one UUID in your .cfg files at a time to unassign a node. If the node is correctly configured & has a good connection it will now appear in the `Found canbus_uuid` report statement by using the above command. If not it wont appear in the return & no unassigned nodes will be visable.

# Using a UUID

Paste the UUID into the selscted MCU section of your .cfg file, making sure the name is what you want. Example below.
```
[mcu EBBCan]
canbus_uuid: 3754cb7564f3
```
Save & restart

The network should now work - in theory!

************************************************

# How tell which of your Canbus nodes are not functional & are keeping the network `Down`.


Your system is set up correctly but your Can network not working, you keep getting can’t connect to MCU’s errors or have communication timeouts during homing….

`ERROR: “mcu.error: mcu 'EBBCan’: Unable to connect”`

First thing to do is go to the m`Machine` Tab in `Mainsail` & download your Klippy Log

Now search using `ctrl+F` or `CMD+F` on MacOS 
```
Starting CAN connect
```
Now use the back arrow to go all the way to the last entry in the file & work up from there. This is the return for the system trying to start the can0 network.

You should see somehting like this, the lines below means the assigned node is active & connected
```
mcu 'mcu': Starting CAN connect
Loaded MCU 'mcu’…..
```
Here is another example for another node on the system, again the assigned node is active & connected
```
mcu 'EBBCan': Starting CAN connect
Loaded MCU 'EBBCan’…..
```
HOWEVER….
This means the assigned node is not responding to the network's attempt to connect & start.
```
mcu 'EBBChamber': Starting CAN connect
mcu 'EBBChamber': Timeout on connect
mcu 'EBBChamber': Wait for identify_response
serialhdl.error: mcu 'EBBChamber': Serial connection closed
mcu 'EBBChamber': Timeout on connect
```
This can either be a psychical connection problem with the Canbus cables, or it’s a firmware related mismatch, or the MCU’s UUID in the .cfg file is incorrect.
************************************************

# ERROR: “Communication timeout during homing”

Make sure your Canbus network frequency is at 100000

If your Can network is running & is at the recommended frequency but you get errors & drop outs you can check the health for each node by going to the “Machine” tab in Mainsail & clicking the name of the connected MCU in the “System Load’s” window. This will bring up a popup window where you can scroll down to the Canbus information & check for numbers on the “bytes_retransmit” &  “bytes_invalid” lines. You can also check the Canbus frequency here too!


![MCU Stats](https://github.com/user-attachments/assets/638763c7-316d-42c7-8f31-0be74cf933af)



If you have anything but 0 for “bytes_retransmit” &  “bytes_invalid” it could mean your Can network might have issues with signal noise &/or problems on the Pi or U2C adapter.

You can check this & more info on SSH by using:
```
ip -details -statistics link show can0
```
This will show the health of your network.


![Canbus Network Status](https://github.com/user-attachments/assets/8db5ff1a-7760-4518-b93d-a1d419670b4f)



NOTE: BTT U2C units might require the BTT fork of Candlelight to work correctly if you’re having issues you can’t resolve. 

To revolve Signal noise on your Can H & L wires be sure to use twisted pair shielded cables & move them away from any high voltage & high current cables. Also try some ferrite chokes, one at each end of the cable, they might help you. 
Ensure all cable connections & plugs are good & not loose or intermittent. This is the prime cause for Canbus drop outs!!

