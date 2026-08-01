# Reverse engineering Tapo C220 (TP-Link)

### Background and introduction
The Tapo C-series camera by TP-Link is widely used and sold worldwide. Some examples are the C200, C210 and C220 cameras. I conducted some research on the Tapo C200 back in 2021, where I also published a series of whitepapers to both Europol's Forensic Experts Forum (FEF) and The European Conference on Cyber Warfare and Security (ECCWS). The paper from ECCWS can be found [here](https://www.researchgate.net/profile/Joakim-Kargaard-2/publication/356750268_IoT_Security_and_Forensics_A_Case_Study/links/61a9fa9faade5b1bf5fea8ad/IoT-Security-and-Forensics-A-Case-Study.pdf). A similar version of the paper was submitted to Europol as well, but with minor changes. It was possible to achieve root access on the Tapo C200 by exploiting a hardcoded root password by using Universal Asynchronous Receiver/Transmitter (UART). If you want to look more into UART, please check out my previous writeup regarding Ebitcam [here](https://github.com/ErikDM/reverse-engineering-ebitcam).

As I have previously conducted plenty of research and hardware hacking on a series of other cameras and routers, it was time to head back to the Tapo C-series. I went to my local shopping mall here in Norway and purchased the Tapo C220 from TP-Link.

<img width="690" height="612" alt="bilde" src="https://github.com/user-attachments/assets/b27b6b46-a8fb-42bc-b377-e8adf124a1b0" />


<br>
<br>

### Disassembling the Tapo C220 and chip extraction

The first step was to disassemble the camera and have a look at the physical components on the curcuit board. We could approach the camera and try to identify the UART pads (if available). However, we are taking another approach this time. There are two reasons for this. The first one is to bypass all security mechanisms and dig into the firmware immediately. The second one is to demonstrate how firmware extraction can be achieved directly from the storage chip. In the upper-right corner you can see the chip where the firmware is stored.

<img width="691" height="805" alt="bilde" src="https://github.com/user-attachments/assets/631a9453-5106-4cb1-997e-51a2dce97084" />


<br>
<br>

This is also known as Serial Peripheral Interface (SPI) flash storage chip. SPI flash storage is a type of non-volatile memory (meaning it retains data even when powered off) that communicates using the SPI protocol. You can find more information on the chip [here](https://www.absunshine.com/en/parts/XM25QH128CHIQ-XMC-5439846). All of the juicy data is stored inside this chip, including its firmware. Before proceeding, I powered on the camera and configured it to connect to my wireless test network. This is to seed relevant data into the camera, which we can search for later (such as wifi password). Also, if you look at the chip, you can see a dot in lower-left corner. This will be useful later when we extract the data in order to know how to connect to the chip.

<img width="1440" height="1920" alt="bilde" src="https://github.com/user-attachments/assets/8220cc8e-a1e8-4c63-89da-4d3d1764a59e" />


<br>
<br>

There are two techniques which can be used to dump out the data using SPI. The first one is to let the chip stay on the board (as shown in the picture above) and connect a SOIC8 clip to it. In the other end, you can connect for example a bus pirate to dump the data. From my experience, this technique may fail and is harder to achieve a successful dump. I have managed to dump the data from different chips several times using this technique, but it can be quite difficult and requires more patience. One reason is that every single connection to the chip needs to be 100% connected, or else the data read will fail. Also, existing components on the circuit board may interfere while attempting to read the data. In practice, you would have to use the existing power supply for the camera to power it up, and then finally conducting the data dump, or you could just use the bus pirate to power it on.

The second technique is to conduct a "chip-off", which means that you extract the chip using a hot air gun. From thereon, you can use the bus pirate to power the chip manually and achieve data extraction once connected to it using a SOIC8 clip (or even solder the wires to the chip). There are many sockets, clips, and other techniques that can be used here. Personally, I just stick with the SOIC8 clip for demonstration purposes. This is a very cheap and practical tool. Everything works eventually... The downside of the chip-off technique is that you would have to solder the chip back to the board afterwards (if you want to power on the device and get it running later on).

In the picture below I am using a hot air gun to extract the chip from the camera. By carefully moving the hot air gun back-and-forth and not having too high temperature, we can prevent damaging the chip and the data that is stored. Also, use flux for easier heat transfer across the component. The picture below is for demonstration purposes before the actual extraction happens.

<img width="1920" height="1440" alt="bilde" src="https://github.com/user-attachments/assets/780c146f-d179-4822-8233-5d5945592151" />


<br>
<br>

After a brief moment, the chip was successfully extracted from the board. Let us clean the chip and bring it back to the lab!

<img width="500" height="653" alt="bilde" src="https://github.com/user-attachments/assets/d4559744-354a-4153-9bae-51def952140d" />


<br>
<br>

### Data extraction using SPI

The next step is to read the data from the chip. Earlier, I mentioned the dot on the chip in order to know how to connect to it before data extraction. The illustration below shows how the chip should be connect 1-1 to the bus pirate. Notice the "dot" in the upper-left corner.


```
         ┌*──────┐
  CS ──▶│1     8│◀── VCC
 MISO ─▶│2     7│◀── HOLD#
 WP# ──▶│3     6│◀── CLK
 GND ──▶│4     5│◀── MOSI
         └───────┘
```


| Flash pin | Flash signal  | Programmer signal      |
| --------: | ------------- | ---------------------- |
|         1 | `CS#`         | `CS`, `SS` or `CS#`    |
|         2 | `MISO` / `DO` | `MISO`, `DO` or `SO`   |
|         3 | `WP#`         | `WP#` or held high     |
|         4 | `GND`         | `GND`                  |
|         5 | `MOSI` / `DI` | `MOSI`, `DI` or `SI`   |
|         6 | `CLK`         | `CLK`, `SCK` or `SCLK` |
|         7 | `HOLD#`       | `HOLD#` or held high   |
|         8 | `VCC`         | Correct supply voltage |


Now that we know what the legs on the chip do and their functions, we can connect these 1-1 to the bus pirate. The bus pirate has its own SPI data read functionality which we can use to dump the data on the chip. Notice the SOIC8 clip that I am using. In practice, I just inserted the chip the correct way according to the illustration above and connected it to the bus pirate.

<img width="1440" height="1920" alt="bilde" src="https://github.com/user-attachments/assets/a6ee1f02-626d-4ba5-a59d-5acc7af6a6e9" />


<br>
<br>

You can also use bus pirate's own converter/socket if necessary. This is a bit more expensive and overall reliable. However, I will continue to demonstrate the data dump using a cheap SOIC8 clip just for the lulz. The same clip can be used to read the chip directly on the board as well.

<img width="716" height="649" alt="bilde" src="https://github.com/user-attachments/assets/4a692cb2-186a-4946-b298-88886598b40a" />


<br>
<br>

Entering SPI-mode on the bus pirate, we can power on the chip (`W` command) using 3.3 volts in this case. You can easily measure this yourself on the board when the power is on. The next step is to probe the chip by utilising the command `flash probe`. If the chip is connected correctly and the power is on, we can now start getting information about the chip and dump data.

<img width="919" height="809" alt="bilde" src="https://github.com/user-attachments/assets/43cdd6e8-2bf7-49a6-ae10-d46353171480" />


<br>
<br>

Entering the command `flash read -f c220.bin` will dump out all data stored on the chip. This might take a few minutes. In return, we get a `c220.bin` file that we can reverse engineer in order to look into the firmware and other data that may be stored.

<img width="1045" height="758" alt="bilde" src="https://github.com/user-attachments/assets/779cb7f3-f04b-4621-b34a-3bff6cdefe69" />


<br>
<br>

After conducting a successful dump, we can transfer the binary to our Kali and start digging into the data using `binwalk`. Keep the offset `0x40100` in mind, as this will be relevant later on in the writeup. We can also see the squashfs filesystem at offset `0x400000`.

<img width="1313" height="761" alt="bilde" src="https://github.com/user-attachments/assets/de13c9aa-41dd-4a40-96c4-884c26fba902" />


<img width="1307" height="325" alt="bilde" src="https://github.com/user-attachments/assets/7eba38cf-56b2-4f13-87e7-8fc060875091" />


<br>
<br>

### Data analysis

Looking at the `squashfs-root` directory, it usually reveals all the details about the operating system, such as `/etc/shadow`, clear-text passwords in config files and much more. This was the case for the C200 camera that I did research on previously. However, there is no relevant or useful data to be seen as this point. All folders contained standard files and configurations, but no personal data, such as my wireless network password, root login credentials (no shadow file to be seen) and so forth. But where is it...? This is where it gets really interesting.

<img width="792" height="258" alt="bilde" src="https://github.com/user-attachments/assets/fab2b3e6-0104-4b70-af4c-e6b46d90faf3" />


<br>
<br>

After doing some research, the camera decrypts the personal data using a key. The data is then used when the camera is on. So where is the data? Well, it is stored somewhere completely different, and not directly inside the operating system (squashfs) with its traditional static files. Digging into the `C220.bin` file, I discovered something called `NVMP-CONFIG`. Non-Volatile Memory (NVM) is the camera's non-volatile memory/configuration partition. The data is stored separately from the operating system and it is encrypted. You cannot access the data without a key. The NVM partition can be found using `xxd` or `strings`. From thereon, you can find the offset to carve it out manually using `dd`.

<img width="840" height="223" alt="bilde" src="https://github.com/user-attachments/assets/1401c56b-41b6-4e0a-89ab-e4a7fc31bc58" />


<br>
<br>

We can now carve out the NVM partition. However, this will still be encrypted. The data cannot be accessed without the key.

<img width="1025" height="137" alt="bilde" src="https://github.com/user-attachments/assets/4bdb2b9b-c9a9-4a02-a385-bbfa72f5f538" />


<br>
<br>

Continuing to search through the SPI data dump, we discovered a few other files in a different location (other than the actual operating system). This also gave us a tarball, which allows us to extract the files. One of these files is called `encrypt_key`. Looking at the key and its usage, I discovered that this can be used to decrypt the NVMP data. This will ultimately give us access to the personal data that is stored inside the camera.

<img width="828" height="507" alt="bilde" src="https://github.com/user-attachments/assets/f1359224-239e-4708-bb28-fcdbe98b0fe0" />


<br>
<br>

Decrypting the `NVMP.bin` file with the key using `openssl` returns a zlib-compressed binary stream. This needs to be decompressed in order to read the files. This can be achieved with either perl or python (zlib library). From thereon, we can create a clear-text data blob from the zlib binary stream. In short terms, this is the flow in order to access and decrypt the data using the key:

1. Binwalk found and extracted data from offset `0x40100` (gzip payload. Look at the binwalk screenshot earlier).
2. Inside was a tarball. Extracting the file `default-tar` gave us the directory `base-files`, which also contained `base-files/etc/encrypt_key`.
3. The file `encrypt_key` kan be used with `openssl` to decrypt the data. This will return a zlib binary stream.
4. We can convert the zlib binary stream to readable data. In this case, this is a giant JSON data blob containing all information from inside the camera.

The next step is to follow this methodology to access the relevant (and juicy data) inside the camera. We will do this step-by-step from scratch.

We start with the original `C220.bin` file that we dumped using SPI. The next step is to carve out the NVMP data, which is encrypted. We name this `nvmp.bin`.

<img width="1028" height="138" alt="bilde" src="https://github.com/user-attachments/assets/81128eb6-f605-4a03-a691-91febefb028b" />

<br>
<br>

Next, we decrypt the NVMP data using `openssl` with the key located at `base-files/etc/encrypt_key`. This will create a zlib binary blob.

<img width="559" height="219" alt="bilde" src="https://github.com/user-attachments/assets/7c20acf9-e1f0-4ec3-9326-bd0ffb8669d2" />

<br>
<br>

Finally, convert the zlib binary blob to readable text using python's zlib library.

<img width="1805" height="81" alt="bilde" src="https://github.com/user-attachments/assets/7a64a8a7-7656-40df-9d65-d6772bb97cd3" />

<br>
<br>

You can create and use a Python script to structure the data and make it more readable, instead of a giant JSON blob. There are in total 189 JSON objects, which can be structured individually. Many of these contains valuable data, epsecially during a forensic investigation. You can find clear-text passwords, username/e-mail, cloud tokens, cloud location/URL, private certificates, general camera settings with triggers, and so much more.

<img width="1205" height="208" alt="bilde" src="https://github.com/user-attachments/assets/642060da-068a-412a-8a68-9735bfb59de6" />

<br>
<br>

After structuring the JSON objects, here are a few examples of the data that was observed on the camera. Clear-text wireless password and other information. This is valuable if the password is re-used or even to access the network itself.

<img width="572" height="576" alt="bilde" src="https://github.com/user-attachments/assets/7deda829-8ab6-4972-86ad-bd92584a4fcd" />

<br>
<br>

I also identified camera credentials and relevant information (root and admin passwords). The camera is also vulnerable to [CVE-2018-11482](https://nvd.nist.gov/vuln/detail/cve-2018-11482), as it uses the exact same password from 8 years ago (`zMiVw8Kw0oxKXL0`). However, when I did my initial research on the Tapo C200 in 2021, I discovered that the root password was `slprealtek`. Seems like they have gone back to one of the old passphrases.

<img width="1185" height="251" alt="bilde" src="https://github.com/user-attachments/assets/f2f5400d-3003-4e41-893d-77a016929cc8" />

<br>
<br>

I also identified information about the cloud account that was used to set up the camera. Two examples are the cloud token used to bind the camera to TP-Link's cloud service, but also the initial username/e-mail that was used to pair/register and join the device.

<img width="854" height="203" alt="bilde" src="https://github.com/user-attachments/assets/a75b58af-a462-41c9-9c66-02880482efac" />

<br>
<br>

The firmware URL download location is also exposed in clear-text.

<img width="885" height="101" alt="bilde" src="https://github.com/user-attachments/assets/39cf9970-0f08-461c-b006-8c030894e8ac" />

<br>
<br>

### Conclusion

There are many differences compared to the Tapo C200 camera I did some research on back in 2021. The overall data is no longer stored in plain sight and could be easily accessed using the traditional operating system, for example cracking `/etc/shadow` and achieving root access within minutes. It seems like there has been an overall upgrade regarding security by using NVM, even though there are flaws in the C220 as well (clear-text key stored in a different location). Armed with the knowledge from this writeup, it would be trivial for a forensic investigator to achieve data extraction within a short period of time. The data stored inside the C220 camera can still be of value to a forensic investigator. There is a lot more data stored inside the JSON objects on the device that I did not include in this writeup.


