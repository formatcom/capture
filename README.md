## Configuración creada para mi canal lowlevel

### Documentación oficial

- https://ffmpeg.org/ffmpeg-all.html
- https://ffmpeg.org/ffmpeg-devices.html#lavfi
- https://trac.ffmpeg.org/wiki/HWAccelIntro
- https://trac.ffmpeg.org/wiki/Hardware/VAAPI
- https://trac.ffmpeg.org/wiki/Hardware/QuickSync


### Crear un device v4l2 loopback para poder tener un preview de la webcam

~~~
$ ffmpeg -y -f v4l2 -s 320x240 -i /dev/video1 -pix_fmt yuyv422 -f v4l2 /dev/video0
~~~

### Abrir preview

~~~
$ ffplay /dev/video0
~~~

### Software decode + h264 qsv encode with 100Mbps using CBR mode

~~~
 Se captura usando el modo CBR es decir bitrate estatico para ahorrar recursos,
 adicional a esto, se estan guardando 4 archivos por separado.

 - console.mp4 [capturadora solo el video, h264_qsv CBR 100Mbps 50 fps]
 - webcam.mp4  [webcam solo el video, h264]
 - console.wav [capturadora solo el audio]
 - mic.wav     [microfono audio contiene ruido dependiendo la calidad del microfono]

$ ffmpeg -y \
	-f pulse -i "alsa_input.usb-USB3.0_HD_Audio_Capture_USB3.0_HD_Video_Capture_HU123450-00.analog-stereo" \
	-f pulse -i "alsa_input.pci-0000_00_1f.3.analog-stereo" \
	-init_hw_device qsv=hw -filter_hw_device hw \
	-f v4l2 -r 50 -video_size 1920x1080 -i /dev/video3 \
	-f v4l2 -i /dev/video0 \
	-filter_complex "hwupload=extra_hw_frames=64,format=qsv" \
	-map 2:v -c:v h264_qsv -b:v 100M -maxrate 100M console.mp4 \
	-map 3:v -c:v h264 webcam.mp4 \
	-map 0:a console.wav \
	-map 1:a mic.wav
~~~

### Eliminar el ruido del sonido grabado con el microfono

~~~
$ sox mic.wav -n noiseprof noise.prof
$ sox mic.wav mic.out.wav noisered noise.prof 0.25
~~~

### Generar video final [modo VBR, es decir variable bitrate]

~~~

Pasos que sigo, para generar el video final

 - Ajustar el volumen de la consola y del microfono.
 - Aplicar filtros para mejorar la calidad de la voz al sonido grabado del mic.
 - Ajustar el sonido del juego con el video del juego para eso utilizo atrim y asetpts.
 - Unir ambos sonidos, es decir el del juego y el del microfono.
 - Superponer el video de la webcam en el video del juego.

$ ffmpeg -y \
	-init_hw_device qsv=hw -filter_hw_device hw \
	-i console.wav -i mic.out.wav -i console.mp4 -i webcam.mp4 \
	-filter_complex "
		[0:a]volume=1.0[a0], [1:a]volume=3.0[a1];
		[a1]highpass=f=200,lowpass=f=3000[snd2],
		[a0]atrim=start=3.7[snd1], [snd1]asetpts=PTS-STARTPTS[gamesnd],
		[snd2]atrim=start=1.3[mic], [gamesnd][mic]amix[sndmix];
		[2:v][3:v]overlay=10:230[vid]; [vid]hwupload=extra_hw_frames=64,format=qsv[ovid]"
	-map "[sndmix]" -map "[ovid]" -c:v h264_qsv -b:v 100M output.mp4
~~~
