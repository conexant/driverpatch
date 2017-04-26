# Conexant AudioSmart 4-Mic Development Kit Driver Porting Guide.

This page provides porting guidelines for Conexant AudioSmart Dev Kit.  

The folder [rpi-patch/](https://github.com/conexant/driverpatch/tree/master/rpi-patch) contains driver patches are based on RPi kerenl branch [rpi-4.4.y](https://github.com/raspberrypi/linux/tree/rpi-4.4.y) with kernel version 4.4.50 or commit id "e223d71ef728c559aa865d0c5a4cedbdf8789cfd"

Please note that the patch might be not applied cleanly if you are using different kernel version. Fot that case, you need to take look at what changes are in the patch, and add them to you kernel manually.

Porting the driver requires the followig steps. 

## 1. Download Download specified RPI kerenl tree :

```
$git clone -n --branch rpi-4.4.y https://github.com/raspberrypi/linux
$cd linux
$git checkout
```

## 2. Roll back to kerenl v4.4.50 [option]

If you want the patch files can be applied cleanly, you can roll back to kernel 4.4.50.
```
$git reset --hard e223d71ef728c559aa865d0c5a4cedbdf8789cfd
```	
## 3. Apply the patches.

```
$git am /path/to/patch/folder/*.patch
```

## 4. Enable audio drivers for Smart Speaker EVK.

load the default setting for RPi3/2
  
```
$KERNEL=kernel7
$make bcm2709_defconfig
$make ARCH=arm menuconfig
```
And then go to menu
```
	[Device Drivers] => [Sound card support] => [Advanced Linux Sound Architecture]
	[ALSA for SoC audio support]=> [Support for Smart Speaker Pi add on soundcard (USB)]
```
Check the following item.
```
	[Support for Smart Speaker Pi add on soundcard (I2S)]
```
## 5. Build kerenl and modules .

Following insturation on the following link to build and insatll kernel
https://www.raspberrypi.org/documentation/linux/kernel/building.md

## 6. Load the overlay DT.

open /boot/config.txt and add the following statemnt.

```
dtoverlay=rpi-cxsmartspk-usb
```


## Change the MCLK and I2S setting.
Both MCLK and I2S setting are specified within ASoC machine driver in the
following path.

```
<Kerenl Tree>/sound/soc/bcm/cxsmtspk-pi-usb.c
```

A) The I2S format can be change by modify the .dai_fmt

```c
          .dai_fmt = SND_SOC_DAIFMT_CBM_CFM |                             
                    SND_SOC_DAIFMT_I2S | SND_SOC_DAIFMT_NB_NF, 
```	
within structure below.

```c
	static struct snd_soc_dai_link cxsmtspk_pi_soundcard_dai[] = {                  
			{                                                                       
					.name = "System",                                               
					.stream_name = "System Playback",                               
					.cpu_dai_name   = "bcm2708-i2s.0",                              
					.platform_name  = "bcm2708-i2s.0",                              
					.codec_dai_name = "cx2072x-dsp",                                
					.codec_name = "cx2072x.1-0033",                                 
					.ops = &snd_cxsmtspk_pi_soundcard_ops,                          
					.init = cxsmtspk_pi_soundcard_dai_init,                         
					.dai_fmt = SND_SOC_DAIFMT_CBM_CFM |                             
							SND_SOC_DAIFMT_I2S | SND_SOC_DAIFMT_NB_NF, 
```
	
Where SND_SOC_DAIFMT_CBM_CFM stand for codec is I2S Master. 
Please change it to SND_SOC_DAIFMT_CBS_CFS if you want the SoC as I2S
Master.
	
							
B) The MCLK frequency is specified by ALSA API below. The default value is
    12.288 MHz 

```c
	snd_soc_dai_set_sysclk(rtd->codec_dai, 1, CX20921_MCLK_HZ,       
                SND_SOC_CLOCK_IN);                                              
```				
The following is code snap of MCLK settings.

```c
	#define CX20921_MCLK_HZ  12288000   
	static int cxsmtspk_pi_soundcard_dai_init(struct snd_soc_pcm_runtime *rtd)      
	{                                                                               
			/* DOTO: Keep the mic paths active druing suspend.                      
			*                                                                      
			*/                                                                     
			struct snd_soc_card *card = rtd->card;                                  
                                                                                
			dev_dbg(card->dev, "dai_init()\n");                                     
                                                                                
			snd_soc_dapm_enable_pin(&card->dapm, "AEC REF");                        
			snd_soc_dapm_sync(&card->dapm);                                         
                                                                                
			return snd_soc_dai_set_sysclk(rtd->codec_dai, 1, CX20921_MCLK_HZ,       
					SND_SOC_CLOCK_IN);                                              
	}                                                                               

```

