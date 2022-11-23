---
layout: post
title:  "RZdalne aktualizacje oprogramowania z narzędziem SWUpdate"
date:   2022-11-22 00:12:24 +0100
categories: jekyll update
---
## List of contents

## A few words at the beginning...
You can image yourself the situation when you had bought a new car and after some time you received the information that you need go to the service because your lovely new car require service action. It's not very bad, a lot of devices don't have possiblity for any update during their lifecycle, but it can be better. What in case when whole proccess will be performed autoamatically and you don't go anywhere and even don't know that something has been changed in your car software. Nowadays software updates 'on-demand' semms to be very important part of product lifecycle. It is better for users and developers when device is updateable in some easy way.

Updates can be categorized according to many different criteria. The first one is update medium. Sometimes from secuirity reasons device can not be online and one way to update it is do that with i.e USB or SDcard. However, if there are no obstacles to be online remote updates can be used. I would like focus in this article on remote updates but many of the issues discussed here will be true for offline updates.

The main aim of article is to discuss the topic of updating embedded devices. The article is in form of tutorial, which presents different ways to update raspberry pi 4 board and by the way disscuss a few aspects and problems related with updates. I didn't want to do everything from scratch and I decided to use of open source framework named SWUpdate. In article topics like will be covered:
* update both entire images and individual applications
* rollback mechanism
* secuirity 
* SWUpdate yocto integration

## What is SWUpdate ?
The aim of the project is to help update an embedded systems from a storage media or from network. It should be considerd as a framework, where handlers or installers, can be easy customizable. The framework has great potential and many options. Not of them will be disscussed here. To get more knowledge about Sproject please vist their website.

The most common usage of SWUpdate is to do image based updates using a “Single Copy” or a “Dual Copy” strategy. SWUpdate does support updating single files on you root file-system or other partitions

#### Single image delivery
The main concept is that the manufacturer delivers a single big image. All single images are packed together (cpio was chosen for its simplicity and because can be streamed) together with an additional file (sw-description), that contains meta information about each single image.

## Prerequisites
- Raspberry Pi Board (I have Raspberry Pi 4B but there will be not many differences for other board version)
- Host computer with MacOS or Linux system ()in tutorial MacOs has been used)
- A litlle technical knowlegde and free time...

## Setup environment
1. Setup environment
- flash a card with OS image (Raspberry Pi OS Lite 64-bit). It is important to prepare image with enabled ssh. I used Raspberry Pi Imager program to do that.
- start board, connect to it  with ssh and install SWUpdate with apt or from sources. In tutorial second opotion has been choosen
{% highlight ruby %}
#install from sources
cd /home/pi
sudo apt-get install lua5.2-dev libssl-dev libconfig-dev libarchive-dev libzmq3-dev libz-dev libcurl4-gnutls-dev libjson-c-dev
#TODO Is this branch needed, error with duplicated case
git clone https://github.com/sbabic/swupdate.git -b 2017.11 && cd swupdate
make test_defconfig
make && sudo make install

#install with apt
sudo apt-get install swupdate
{% endhighlight %}

- create key pair (if enabled support for signed images) (on host)
{% highlight ruby %}
openssl genrsa -out swupdate-priv.pem
openssl rsa -in swupdate-priv.pem -out swupdate-public.pem -outform PEM -pubout
{% endhighlight %}

- copy generated public key on the board
{% highlight ruby %}
scp swupdate-public.pem  pi@raspberrypi.local:/home/pi/data/security/
{% endhighlight %}

- start SWUpdate (on embedded device)
{% highlight ruby %}
sudo swupdate -v -k /home/pi/data/security/swupdate-public.pem -w "-document_root /home/pi/swupdate/www"
{% endhighlight %}

Since we built SWUpdate with the web-server enabled we can now go to our browser and go to:
http://raspberrypi.local:8080/, and we will be presented with this: 

....

## Update single file
As first update scenario I would like to consider update of single application. This kind of update is not recommended way but it can be used in some special cases. The main benefits are small size in comparision to whole OS image update and it not require to restart of device OS to aplly changes. The problem can arise when after start new software OS will crash. In that situation we cannot have mechanism to restore it in easy way i.e online.
 
### Steps to reproduce
- create sw-description file
{% highlight ruby %}
software =
{
    version = "1.0.1";
    raspberrypi = {
        hardware-compatibility: [ "1.0" ];
        files: (
            {
                filename = "SWUpdate";
                path = "/tmp/SWUpdate";
                sha256 = "27ebafbe58603437d819cde3470811e5e64a8aa876a4fa680e5266e36bbee519";
            }
        );
    };
}
{% endhighlight %}

- create upload file
{% highlight ruby %}
echo "SWUpdate v1" > SWUpdate
√ repos % cat ~/workspace/SWUpdate 
SWUpdate v1
{% endhighlight %}

- sign the package
{% highlight ruby %}
openssl dgst -sha256 -sign ~/swupdate-priv.pem sw-description > sw-description.sig
{% endhighlight %}

- Create an cpio archive (not worked in my case on MacOS, I did commands from this point on raspberry board)
{% highlight ruby %}
export FILES="sw-description sw-description.sig SWUpdate"

for i in $FILES; do echo $i; done | cpio -ov -H odc >  update-image-v1.swu
sw-description
SWUpdate
2 blocks
{% endhighlight %}

- Update board with via webinterface. File to upload: update-image-v1.swu
![image-title](/assets/images/SWUpdate_success.png)

- Verify updated file
{% highlight ruby %}
pi@raspberrypi:~ $ sudo cat /tmp/SWUpdate 
SWUpdate v1
{% endhighlight %}

As you can see one step above,file has been updated. As additional task you can try to upload package with wrong signagurea nd see what happend in that scenario ;)

## Update whole OS image
OS image for embedded device can be often considered as single applications whose individual parts cannot be changed. As example you can image the situation when you provided image for embedded device and later with OS can be updated with help of apt-get command. After updates performed this way OS is different than initial delivered image. Developer can not guarantee that this new image will be work properly becuase it was never tested in that configuration. Maybe after update of some application whole device will not work as expected. Searching of bugs in this kind of customized original image can be nightmare because we never know with which configuration we work.      
Because that reasons the common way for embedded devices is to do update whole image instead of particular files/applications. Main disadvantage of those solution is update size. Whole image is normally bigger than one file/app.

We can distinguish the following types of OS image updates:
- single copy
- dual copy
- diffrential way

In this tutorial i would like to focus on dual copy mode only.

### Dual copy update
In dual copy update the device partitioning look like this:

| BOOTLOADER | OS (active) | OS (inactive) | PERSISTENT DATA |

Can be noticed that we have two OS. One is active and the second one is inactive.
Dual copy update in shortcut contains following steps:
- checking for new updates
- download update to inactive OS
- reboot device and mark inactive OS (updated) as active OS

Now we will pass through simple example of dual copy mode.

git clone git://git.yoctoproject.org/poky -b rocko
git clone https://github.com/openembedded/meta-openembedded.git -b rocko
git clone https://github.com/sbabic/meta-swupdate -b rocko
git clone https://github.com/sbabic/meta-swupdate-boards.git -b master
git clone https://github.com/agherzan/meta-raspberrypi.git -b rocko

greadlink -f layers/poky/oe-init-build-env build
brew install coreutils




