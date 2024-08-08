---
layout: post
title:  "SWUpdate, about software updates in the Embedded Linux world"
date:   2024-08-08 13:00:00 +0100
categories: jekyll update
---

## A few words to start with...
In this article, I would like to discuss the topic of remote updates of Linux Embedded devices. Basic software update strategies will be discussed and the SWUpdate tool will be presented. The article includes examples of simple updates using the Raspberry Pi 4 board and the SWUpdate framework. The motivation for writing the article was my previous experience with updating embedded devices, which often came down to executing scripts written earlier by programmers. As an alternative to this approach, we can use the ready-made, well-tested SWUpdate tool.

Sometimes, for various reasons, the Linux Embedded device does not have a connection to the Internet. In such a case, it is not possible to perform the update remotely, but you have to use a local interface (UART, SD). In the examples, I will focus on remote updates, although in the case of most of the considerations raised, the transmission medium used for the update is not of major importance.

## Software Update Strategies
### Single Application Update
It involves updating a single application. The easiest way to perform this type of update is, for example, using the `scp` command and replacing the application file/files on the disk.

First, I would like to start with the advantages of this solution. The first benefit is the smaller size of the update compared to updating the full system image. The second is that usually after the update it is enough to restart the given application and not the entire operating system for the changes to be noticed.

After updating a single application, we are essentially dealing with a new version of the entire software. The new version of the application may not work correctly with the entire system like the previous version. This approach can quickly lead to problems with dependencies and software inconsistency. Another risk is "breaking" the device. If, for example, the power goes out during the update, the device may end up in an unspecified state and we may lose the possibility of updating again and fixing the situation, at least remotely.

However, the strategy of updating a single application is often sufficient and, above all, fast and easy to use. In the case of projects in the early stages of development, where reliability is not key and the device is often at hand, this approach works very well.

### Full System Image Updates
Another update strategy is to update not a single application, but the entire system. By providing a full system image, we do not have to worry about software consistency, because we provide software in a given configuration that has been previously tested. Another advantage is that if the update fails, or if the power goes out during the update, there are mechanisms available to help us get out of trouble. Its disadvantage is the size of the transferred package. This can be a problem especially when data transfer is limited. This approach also requires more knowledge (bootloader, etc.).

We can distinguish three different strategies here:
- single copy,
- dual copy,
- differential update.

#### Dual copy
Device partitioning looks like this:

| Bootloader | Primary OS (active) | Primary OS (inactive) | Data Partition |

In the case of the dual copy strategy, we have two partitions containing the operating system. At a given moment, only one OS is active, the other remains inactive.

The principle of the update is simple. The device asks for available updates. When it receives a positive response, it downloads a new image and saves it to the inactive partition. In the next step, the device must be restarted with a new saved partition marked as active.

This way we have reached the advantages of this approach. In the case when the newly saved system image does not start correctly, you can always return to the previous version and make it active. When the power goes out during the update, the device will still be able to use the "old" system image and try to update again. Another advantage is that we do not need any interaction with the user, because all operations related to the update can be performed in the background.

The disadvantage of the solution is the amount of space needed. We can say that we need at least twice as much disk space as in the case of updating a single application on a device with a single OS. A certain inconvenience may also be that the device must be restarted.

#### Single Copy
Partitioning the device looks like this:

| Bootloader | Primary OS (active) | Recovery OS | Data partition |

The main difference between the single copy and dual copy strategy is that in the case of the former, we use a Recovery OS instead of a second partition with a full OS. The Recovery OS is a minimal system image from which the device can be booted and provides the ability to install updates.

Updates can be divided into the following steps. The device asks for available updates and if it finds one, it downloads the image and saves it to the data partition. Then it asks the user to start installing the update. In the next step, the device is booted with the Recovery OS, and the previously downloaded image overwrites the Regular OS. Finally, all that remains is to boot the device by loading the primary system image.

Unfortunately, there are a few drawbacks to this approach. We need a lot of space on the data partition because the image has to be downloaded somewhere. The device remains inactive while the update is being installed. In the event of an unsuccessful update, we remain with the device in recovery mode, but on the bright side, we can try to install the updates again. Unfortunately, we can't download the image again because this step is performed using the primary OS and not recovery. In this case, if the previously downloaded image is somehow corrupted, we will have to attempt to manually repair the problem.

#### Differential update
This is a type of update that solves the problem with its size. It consists of downloading and saving the differences between the old and new image. I will not elaborate here because it is a large topic. For more information, please visit: [link](https://sbabic.github.io/swupdate/delta-update.html).

## What is SWUpdate
SWUpdate is a 'Linux Update agent', whose main task is to help update the Embedded Linux system. It can be installed using the package manager command or built from source. The project offers many possibilities and thanks to it we do not have to invent everything from scratch.

Main functionalities:
- support for multiple update strategies, single applications and entire partitions,
- support for multiple data carriers using different handlers (eMMC, SD, Raw NAND, NOR, ...),
- support for multiple local interfaces for downloading software (USB, SD, SPI, UART),
- ability to perform remote updates,
- integrated web server for performing updates,
- continuous query mode for available updates,
- possible integration with servers for managing updates. The current version supports the Hawkbit server,
- support for encrypted/signed updates,
- checking compatibility between software revision and device,
- ability to add your own handlers to support non-standard protocols.

Only some of the possibilities offered by the SWUpdate framework have been listed. For more information, please visit the project's website: [link](https://sbabic.github.io/swupdate/swupdate.html)

### Building an update file
One of the assumptions of the SWUpdate project is to use a single file for updating.
<p align="center">
  <img src="/assets/images/swupdate/swupdate.drawio.svg" />
</p>

- sw-description - contains meta information about the update file. The update file can contain multiple images/files. SWUpdate uses the "libconfig" library as the default parser for the sw-description file. However, it is possible to extend SWUpdate and add your own parser, based on a different syntax and language than the one supported by libconfig,
- image 1 ... image n - individual images/files used during the update.

All individual images/files are packed together with the sw-description file as a `cpio` archive (`cpio` was chosen for its simplicity and streaming capabilities).

## Examples
### Requirements
To complete the examples, we need a Linux device and a computer from which we will update the device. In my case, I used a Raspberry Pi 4 board and a computer with Ubuntu 22.04 installed.

### Preparing the environment
1. Preparing the Raspian system image. It is important that the `ssh` service is enabled by default. [host]
2. Saving the image to an SD card and mounting it on the board. [host, Raspberry Pi]
3. Installing the SWUpdate tool on the device whose software will be updated. SWUpdate can be delivered using a package manager or built from source. The second option was chosen in the example. [Raspberry Pi]
    ```
    #install with apt
    sudo apt-get install swupdate

    #install from sources
    cd /home/pi
    sudo apt-get install lua5.2-dev libssl-dev libconfig-dev libarchive-dev libzmq3-dev libz-dev libcurl4-gnutls-dev libjson-c-dev
    git clone https://github.com/sbabic/swupdate.git && cd swupdate
    make test_defconfig
    // Optionally: make menuconfig
    make && sudo make install
    ```
4. Preparing a key pair (if support for signed packages has been enabled earlier). [host]
    ```
    openssl genrsa -out swupdate-priv.pem
    openssl rsa -in swupdate-priv.pem -out swupdate-public.pem -outform PEM -pubout
    ```
5. Copying the public key to the board. [host]
    ```
    scp swupdate-public.pem pi@raspberrypi.local:/home/pi/data/security/
    ```
6. Running SWUpdate. [Raspberry Pi]
    ```
    sudo swupdate -v -k /home/pi/data/security/swupdate-public.pem -w "-document_root /home/pi/swupdate/www"
    // -v - ustawia maksymalny poziom logowania
    // -k - wskazuje na lokalizacje klucza publicznego, który zostanie użyty do weryfikacji instalowanej aktualizacji
    // -w - mówi, że zostanie uruchomiony wbudowany webserwer
    // -document_root - ścieżka gdzie znajduje się `document root` webservera. Jeśli SWUpdate został zainstalowany za pomocą menadżera pakietów to trzeba pobrać repozytorium projektu, ponieważ w nim znajduje się wymagany "document root" dla webservera 
    ```
7. As the last step to verify, we can launch the browser and enter the SWUpdate web interface page: http://IP-URZĄDZENIA:8080. [host]
![image-title](/assets/images/swupdate/SWUpdate_start.png)

### Example 1 - single application updates
#### Update, step by step
1. Create sw-description file. [host]
    ```
    software =
    {
        version = "1.0.1";
        raspberrypi = {
            hardware-compatibility: [ "1.0" ];
            files: (
                {
                    filename = "myApp";
                    path = "/usr/bin/myApp";
                    sha256 = "a25c88c564ad0e8c42d9863c3a4309c88a264a4b2fca56031e835f05a8e653d1";
                }
            );
        };
    }
    ```
Parameters description in sw-description:
version: version to which we are updating,
filename: name of the application to be updated,
path: path to the application being updated,
sha256: hash of the new application version.

2. Signing the package. [host]
    ```
    openssl dgst -sha256 -sign ~/swupdate-priv.pem sw-description > sw-description.sig
    ```
3. Creating a cpio archive, in my case I do it in Bash. [host]
    ```
    FILES="sw-description sw-description.sig myApp"

    for i in $FILES; do echo $i; done | cpio -ov -H odc >  update-image-v1.swu
    sw-description
    sw-description.sig
    myApp
    3 blocks
    ```
4. Updating the board using the web interface. This step requires launching a browser and launching the SWUpdate web interface. The address of my Raspberry Pi 4 board is as follows: http://raspberrypi.local:8080. Then, using the button on the page, we point to the update file: update-image-v1.swu [host]
![image-title](/assets/images/swupdate/SWUpdate_success.png)
5. To verify that the application has been updated correctly, I connect to the board using `ssh` and check whether the contents of the application file have been replaced with the new version. Previously, the myApp file contained the text `myApp v0` [Raspbery Pi]
```
pi@raspberrypi:~ $ sudo cat /usr/bin/myApp
myApp v1
```
The application has been successfully updated. As an exercise, I suggest sending an update package without or with an incorrect signature and see what message we get.
 
### Example 2 - hardware compatibility
One of the options that the SWUpdate framework allows is to check the compatibility of the update with the hardware on which it is to be installed.

1. On the Linux Embedded device, you need to create the /etc/hwrevision file, the path may be different, but then you need to tell SWUpdate about it during startup via a parameter. This file describes the hardware revision. On my device, the file looks like this:
```
pi@raspberrypi:~ $ cat /etc/hwrevision
raspberrypi 1.0
```
2. The next step is to add the revision- hardware compatibility in the sw-description file, for which the update is intended:
```
software =
{
    version = "1.0.1";
    raspberrypi = {
        ///software compatible with hardware
        hardware-compatibility: [ "1.0" ];
        // incompatible software with hardware
        // hardware-compatibility: [ "3.0" ];
        ...
    }
    ...
}
```
If the update is not compatible with the hardware revision, it will not be installed and will return an error message:
![image-title](/assets/images/swupdate/SWUpdate_hwrevision.png)

## Summary
The SWUpdate framework allows you to approach updates in a simple and safe way. It is an extensive tool and has many other interesting options, such as updates using a full system image or integration with the HawkBit server, which I recommend familiarizing yourself with.
