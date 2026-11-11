# ASUS ZenBook 13 OLED UM325U (Ryzen 5 5500U) — Hackintosh con macOS Tahoe 26.7.3

![asus-zenbook-14_um325ua-kg189w_06](https://github.com/anerik70/Asus-Zenbook-13-UM325U/assets/5209138/6566b8de-25c6-4ec8-b32f-beb3cf952c54)

Un Hackintosh portátil Ryzen (Cezanne) funcional con **macOS Tahoe 26.7.3 (25G416)**, actualizado en noviembre de 2026. El resultado principal: **el control nativo de brillo del OLED funciona** — resuelto a través de la interfaz de retroiluminación AUX de DisplayPort de AMD con [AuxDim](https://github.com/nickk19/AuxDim) 0.3.0, sin atenuación por software. El sueño sigue siendo el único frente sin resolver, y esta página documenta exactamente dónde y por qué falla, para que nadie más tenga que deducirlo desde cero.

> Este repositorio es anterior a Tahoe (empezó en 2023 con un macOS más viejo). Todo lo que sigue describe la construcción **actual**. Datos de hardware que cambiaron desde la primera versión de este README: la iGPU es una **Vega 7** (Cezanne, no Renoir/Vega 10), el códec de audio es un **ALC295** y el SSD ahora es un **WD_BLACK SN850X de 1 TB**.

## Hardware

| Pieza | Detalle |
|---|---|
| CPU | AMD Ryzen 5 5500U (Cezanne) |
| RAM | 16 GB LPDDR4X |
| GPU | AMD Radeon RX Vega 7 (0x164C), vía NootedRed |
| Pantalla | OLED Samsung **ATNA33XC11-0**, 13.3" |
| Wi-Fi / BT | Intel AX200 |
| Audio | Realtek ALC295 |
| Trackpad | ELAN1206, I2C |
| Teclado | PS2 |
| Lector de tarjetas | Realtek (PCIe, rtsx) |
| Almacenamiento | WD_BLACK SN850X 1 TB (el NVMe ASUS de fábrica fue reemplazado) |
| Cámara | Cámara web HD USB (IR, Windows Hello) |

Arranque múltiple en el mismo SSD: macOS + Windows + Kali Linux, cada uno con su propia partición EFI.

## Pila de software

- **OpenCore 1.0.8**, SMBIOS `MacBookPro16,2`
- **Lilu 1.7.1**
- **NootedRed 0.9.0** (aceleración de la iGPU, Metal funciona)
- **AirportItlwm 2.4.1** (Wi-Fi; sin parches raíz de OCLP — fueron revertidos y no se usan)
- **AppleALC 1.9.5 + SimpleHDA**, layout `alcid=44`
- **Mapa USB personalizado** construido en Tahoe con USBMap de corpnewt (controladores XHC0 + XHC2; puertos de la cámara y el Bluetooth marcados como internos, conector 255)
- **Bluetooth Intel**: IntelBluetoothFirmware + IntelBTPatcher 2.5.2 (fork de lshbluesky) + BlueToolFixup 2.7.2
- **AuxDim 0.3.0** — brillo del OLED por la interfaz AUX de AMD (detalles abajo)
- **AMDMicrophone** para el micrófono interno (instalado en /Library/Extensions)

Boot-args de producción:

```
-v amfi=0x80 -ibtcompatbeta debug=0x144 keepsyms=1 alcid=44 revpatch=sbvmm,cpuname itlwm_cc=CO -lilubetaall -auxdimamd
```

(El trío `-v debug=0x144 keepsyms=1` son restos de depuración que mantengo a propósito; una instalación limpia puede quitarlo.)

## ⭐ Brillo nativo del OLED — RESUELTO (AuxDim, backend AMD)

Este fue el problema más largo de esta máquina. En Tahoe la ruta clásica de retroiluminación PWM está muerta: el slider existe, macOS envía las peticiones (PNLF/AppleBacklight están presentes) y el panel las ignora. El modo HDR *sí* cambia la luminancia pero titila (ver Problemas conocidos), así que durante más de un año la respuesta práctica fue la atenuación por software con Lunar.

La solución llegó con [AuxDim](https://github.com/nickk19/AuxDim), un plugin de Lilu que atenúa paneles OLED por el canal AUX de DisplayPort. Lo que encontramos en este panel, y que puede servirle a otros dueños de equipos Cezanne/OLED:

- El framebuffer de NootedRed **sí** publica una interfaz AUX nativa en AMD (`native-aux=yes`) — el primer dato AMD confirmado para el proyecto.
- El panel anuncia **ambas** interfaces de retroiluminación, pero el firmware de ASUS no activa ni el modo nits de Intel (DPCD `0x344 = 0x00`) ni el modo VESA (`0x720` bit 0 = 0).
- Volcados DPCD en vivo tomados en Linux (Kali, `/dev/drm_dp_aux0`) con el brillo a ~20% y al 100% probaron que el panel se maneja en realidad por la **interfaz de nits propia de AMD**: habilitación `0x32F = 0x01`, nivel en `0x320` en **milinits** (80.000 → 400.000), lectura de vuelta reflejada en `0x326`/`0x32A`, tiempo de transición en `0x324` (50 ms) y control en `0x32E`. El registro de nivel de Intel, `0x354`, también sigue el brillo (80 → 400); el bloque VESA `0x700` nunca cambia.

Con esos volcados, nickk19 construyó un backend AMD en tres días. Desde **AuxDim 0.3.0**:

- El boot-arg **`-auxdimamd`** junto con el slider de macOS (y las teclas Fn) manejan el panel directamente por AUX — atenuación real del panel, no oscurecimiento por software.
- La posición más baja del slider apaga el panel por completo (el piso por defecto es 0 nits; `auxdimminnits=N` mantiene un mínimo visible si lo prefieres).
- El brillo se conserva tras dormir y despertar la pantalla (verificado: cientos de escrituras AUX, cero fallos).

La historia completa, la salida de la sonda y los volcados: [issue #1 de AuxDim](https://github.com/nickk19/AuxDim/issues/1) — esta máquina fue el primer caso de prueba AMD. AuxDim 0.4.0 (noviembre de 2026) agrega la atenuación a la mitad al estilo macOS justo antes de que la pantalla duerma; la versión verificada aquí es la 0.3.0.

## Bluetooth — funciona (con una receta específica)

El Bluetooth Intel en Tahoe fue el desvío más largo de esta construcción: con varias combinaciones de firmware/patcher el firmware sí se subía a la AX200, pero el `IOBluetoothHostControllerUSBTransport` de Apple nunca se registraba, dejando un controlador fantasma (dirección NULL, firmware v0). Lo que finalmente lo hizo emparejar y mantenerse emparejado:

1. Orden de los kexts al **final** de Kernel → Add: **IntelBluetoothFirmware → BlueToolFixup → IntelBTPatcher** (BlueToolFixup 2.7.2; firmware/patcher 2.5.2 del fork de lshbluesky).
2. Las dos variables de Bluetooth presentes en **NVRAM → Add**, en cero (`bluetoothExternalDongleFailed`, `bluetoothInternalControllerInfo`).
3. Una limpieza de preferencias: `sudo pkill bluetoothd`, borrar `com.apple.bluetooth*` de `/Library/Preferences` y reiniciar.

Por separado, ninguno de los tres pasos bastó (la limpieza de plist ya se había probado semanas antes sin efecto); funcionó la combinación. Una advertencia más de la historia de esta máquina: un binario `/usr/sbin/bluetoothd` corrupto (hash mismatch de datos comprimidos en APFS, visible en la información ktriage del reporte de caída) produce cierres al emparejar que se ven exactamente como un problema de kexts. Si el emparejamiento mata el Bluetooth, lee el `.ips` antes de cambiar kexts.

## Sueño — NO funciona, y aquí está el mapa completo

El sueño es el único frente abierto. Todo lo que sigue fue probado y medido en esta máquina:

- **Por defecto el firmware expone solo S0 Modern Standby + Hibernar** (confirmado con `powercfg /a` en Windows: S3 «el firmware del sistema no admite este estado de espera»). macOS duerme vía S3, así que de fábrica la máquina solo apaga la pantalla.
- **El S3 se puede habilitar en el BIOS.** Con el S3 activo, macOS entra en sueño *real* (LED respirando) — y **se reinicia al despertar, sin pánico y sin reporte «Sleep Wake failure in EFI»**. El DSDT muestra por qué esto es una historia de firmware: el paquete S3 existe pero está escondido como `Name (XS3, …)` y solo se publica como `\_S3` cuando la opción del BIOS está activa; `_WAK`/`_PTS` están completos. La reanudación muere dentro de la ruta S3 del firmware, antes de que el kernel recupere el control. (Efecto secundario: con el S3 habilitado en el BIOS, Windows no arranca.)
- **Las rutas de hibernación son callejones sin salida aquí**: `hibernatemode 25` es ignorado en silencio por la plataforma (la imagen se crea, nunca se usa, no hay entrada de sueño en el log), y HibernationFixup 1.5.4 no cambia nada — la petición de dormir muere antes de que el kext pueda ayudar. `darkwake=0` tampoco hace diferencia.
- **El Deep Idle (S0ix) no puede funcionar en AMD bajo macOS.** [S0ixStandby](https://github.com/labrouss/S0ixStandby) (SSDT-DEEPIDLE + kext) fue instalado y probado: el PEP de esta máquina (`\_SB.PEP`, AMDI0005) está presente con la superficie _DSM LPS0 correcta (máscara Microsoft 0x1F9), el Deep Idle se activó, el LED respiró — y el despertar se colgó con la CPU detenida, idéntico con `-s0ixnodsm` (sin una sola llamada _DSM), lo que absuelve al intercambio kext/firmware. El autor del proyecto confirmó la causa raíz: en el primer Deep Idle, XNU detiene la CPU por una ruta que solo maneja el XCPM de Intel, así que en AMD nunca vuelve. Su v0.5.0 trae una «guarda AMD» basada en el reporte de esta máquina: [issue #2 de S0ixStandby](https://github.com/labrouss/S0ixStandby/issues/2).

Aplicada y conservada (inofensiva, y necesaria si aparece una ruta futura): la receta `pmset` — `hibernatemode 0 standby 0 autopoweroff 0 powernap 0 proximitywake 0 tcpkeepalive 0 womp 0 ttyskeepawake 0`.

**Conclusión práctica para otros dueños de portátiles Ryzen:** si tu firmware esconde el S3 y solo ofrece Modern Standby, espera exactamente este mismo muro en macOS. Apaga el equipo para un descanso real; el sueño de pantalla no es sueño.

## Lo que funciona

- Aceleración gráfica (Metal), HDMI
- **Brillo nativo** (AuxDim, interfaz AUX de AMD) — slider + teclas Fn
- Wi-Fi (AirportItlwm), Bluetooth (emparejamiento incluido)
- Audio (parlantes), teclas de retroiluminación del teclado, teclas de sonido
- Trackpad I2C con gestos, teclado PS2
- Puertos USB (USB-A + ambos USB-C), mapa personalizado
- Cámara integrada
- Estado de la batería, gestión de energía de la CPU, monitoreo de temperatura
- Lector de tarjetas (microSD)

## Lo que no funciona / problemas conocidos

- **Sueño** (ver el mapa completo arriba)
- **Titileo con HDR**: con el HDR activado el panel titila — un rasgo conocido del driver NootedRed, no un daño del panel. En seguimiento aguas arriba: [NootedRed #301](https://github.com/ChefKissInc/NootedRed/issues/301) y [#379](https://github.com/ChefKissInc/NootedRed/issues/379); mi propio reporte anterior es el issue #2 de este repositorio. **Mantén el HDR apagado**; con AuxDim, el control de brillo en SDR está completo de todos modos.
- **Micrófono interno**: funciona vía AMDMicrophone pero con mala calidad — un techo conocido de Cezanne, no un error de configuración.
- **macOS 27 nunca correrá en esta máquina**: es solo para Apple Silicon, sin ninguna compilación x86. Tahoe 26 es el fin de la línea para los Hackintosh x86; se esperan actualizaciones de seguridad por unos tres años más.

## Configuración del BIOS

- Desactiva Fast Boot y Secure Boot
- Deja la opción S3 **apagada** (por defecto) salvo que estés experimentando — con ella activa, macOS se reinicia al despertar y Windows no arranca

## Nota de método

Cada cambio de EFI en esta máquina se hizo de uno en uno, con un respaldo completo y verificado de la EFI antes, una USB de rescate booteable a la mano y una verificación de huella de la partición EFI destino antes de escribir (la numeración de discos de este portátil cambia entre arranques, y la ESP de un disco externo puede verse idéntica — no pregunten cómo lo sé). Si adaptas algo de esta página, mantén la misma disciplina.

## Créditos

- Apple por macOS
- Los desarrolladores de AMD-OSX por los parches de kernel para CPU AMD
- Acidanthera por OpenCore y la mayoría de kexts usados aquí
- [nickk19](https://github.com/nickk19) por AuxDim y por construir el backend AMD a partir de los volcados DPCD de esta máquina
- [labrouss](https://github.com/labrouss) por S0ixStandby y por documentar el veredicto del Deep Idle en AMD
- ChefKissInc por NootedRed
- Trulyspinach por la gestión de energía y el monitoreo para Ryzen
- DhinakG / corpnewt por USBToolBox / USBMap
- Dortania por las guías de OpenCore
- La comunidad AMD-OSX
