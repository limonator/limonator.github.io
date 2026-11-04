---
title: "Reversing del firmware de Espresso"
date: 2026-11-04 
excerpt: "Analizando un binario de Firmware para ESP32"
header:
    overlay_image: /assets/images/ctfs/espresso/banner.jpg
    overlay_filter: 0.5 
    teaser: /assets/images/ctfs/espresso/banner.jpg
classes: wide
---
# Objetivo
Este **challenge** de Hack The Box nos otorga un firmware en un archivo `firmware.bin` y a partir de él, mediante ingeniería inversa tenemos que hallar una flag entendiendo cómo funciona el programa.
# Análisis del archivo
Comenzamos listando las cadenas legibles del archivo mediante el comando `strings` y obtenemos una lista muy grande. Pipeamos a less:
```shell
strings firmware.bin | less 
```
Y comenzamos a leer y encontrar datos interesantes. Me han llamado la atención cadenas que hacen referencia a ESP:
```
esp_ptr_in_diram_dram((void *)dend) || esp_ptr_in_rtc_dram_fast((void *)dend)
-------------------------------------------------
ESP_SLEEP_RTC_USE_RC_FAST_MODE
-------------------------------------------------
//IDF/components/esp_hal_uart/esp32/include/hal/uart_ll.h
```
Entre muchas otras. Eso nos puede indicar que el firmware es para un ESP32. Si buscamos en las cadenas legibles por algo relacionado a la *flag* obtenemos:
```
!((vd->flags & VECDESC_FL_SHARED) && (vd->flags & VECDESC_FL_NONSHARED))
E (%lu) %s: No free interrupt inputs for %s interrupt (flags 0x%X)
flag did not generate correctly.
```
Veamos qué hay en el resto de líneas hacia arriba y abajo:
```
❯ strings firmware.bin | grep "flag did not generate correctly" -A 10 -B 10
esp_vfs_register_fs("/dev/uart", &s_vfs_uart, ESP_VFS_FLAG_STATIC, NULL)
/IDF/components/esp_driver_uart/src/uart_vfs.c
uart
E (%lu) %s: %s(%d): uart_num error
E (%lu) %s: %s(%d): uart driver error
vfs != NULL
//IDF/components/vfs/vfs.c
strncmp(src_path, vfs->path_prefix, vfs->path_prefix_len) == 0
/dev/null
I (%lu) %s: %s
flag did not generate correctly.
It seems you are running the firmware on cloned hadware. <--------- INTERESANTE
E (%lu) %s: %s
Buy the real hardware, or perhaps try to emulate it. ;) <---------- MUY INTERESANTE
Unhandled interrupt %d on cpu %d!
flash_hal
```
El propio firmware nos da la pista de que debemos emular o comprar el software real. Por tanto eso del **ESP32** tiene bastante sentido.
# Subiendo el firmware
Ya que poseo un **ESP32** puedo probar a subir el firmware a ver qué hace el programa. No le hace gracia al firmware mi versión china del ESP:
```
E (260) main: It seems you are running the firmware on cloned hadware.
E (260) main: Buy the real hardware, or perhaps try to emulate it. ;)
I (260) main_task: Returned from app_main()
```
Por tanto lo emularé usando qemu, pero la versión xtensa.
```
qemu-system-xtensa -nographic -machine esp32 \
    -drive file=firmware.bin,if=mtd,format=raw \
    -serial mon:stdio
```
Y obtenemos la flag!!!!

# Conclusión
Ha sido la primera vez que he analizado un firmware, esperemos que sea la primera de muchas ya que es un tema increíblemente útil para hardware hacking. Un gusto resolver otro challenge de HTB como siempre!.
