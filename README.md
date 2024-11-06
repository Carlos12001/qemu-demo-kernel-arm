# Tutorial Completo: Crear y Ejecutar un Sistema Linux Minimalista en QEMU ARM

Este tutorial te guiará paso a paso para construir y ejecutar un sistema Linux minimalista en una máquina virtual ARM utilizando QEMU. Seguiremos el enfoque del tutorial "Minimalistic Linux system on QEMU ARM", pero actualizaremos los pasos para evitar errores comunes y asegurarnos de que todo funcione correctamente.

## Paso 1: Preparar el Entorno

### 1.1. Instalar Dependencias Necesarias

Asegúrate de tener instaladas las siguientes herramientas y dependencias:

```bash
sudo apt-get update
sudo apt-get install -y qemu-system-arm gcc-arm-linux-gnueabi build-essential libncurses5-dev bc bison flex libssl-dev libelf-dev
```

- **qemu-system-arm**: Emulador de sistemas ARM.
- **gcc-arm-linux-gnueabi**: Compilador cruzado para ARM.
- **build-essential**: Herramientas básicas de compilación.
- **Otras dependencias**: Necesarias para compilar el kernel y BusyBox.

## Paso 2: Verificar el Compilador Cruzado

Vamos a comprobar que el compilador cruzado funciona correctamente.

### 2.1. Crear un Programa de Prueba

Crea un archivo llamado `main.c` con el siguiente contenido:

```c
int main() {
    return 0;
}
```

### 2.2. Compilar el Programa

Compila el programa utilizando el compilador cruzado:

```bash
arm-linux-gnueabi-gcc main.c -o test
```

### 2.3. Verificar el Ejecutable

Verifica que el ejecutable es para ARM:

```bash
file test
```

Deberías ver algo como:

```
test: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), ...
```

## Paso 3: Descargar y Preparar el Kernel de Linux

### 3.1. Descargar el Código Fuente del Kernel

Descarga la versión estable del kernel de Linux (por ejemplo, la versión 6.5.7):

```bash
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.5.7.tar.xz
```

### 3.2. Extraer el Código Fuente

Extrae el archivo descargado:

```bash
tar xf linux-6.5.7.tar.xz
```

## Paso 4: Configurar y Compilar el Kernel para ARM

### 4.1. Entrar en el Directorio del Kernel

```bash
cd linux-6.5.7
```

### 4.2. Configurar el Kernel para la Placa VersatilePB

```bash
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- versatile_defconfig
```

### 4.3. Compilar el Kernel

Compila el kernel y genera la imagen `zImage`:

```bash
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- zImage -j$(nproc)
```

### 4.4. Compilar los Device Tree Blobs (DTBs)

Compila todos los DTBs, incluyendo `versatile-pb.dtb`:

```bash
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- dtbs
```

### 4.5. Verificar la Ubicación de `versatile-pb.dtb`

El archivo `versatile-pb.dtb` se encuentra en:

```
ls arch/arm/boot/dts/arm/versatile-pb.dtb
```

## Paso 5: Descargar y Preparar BusyBox

### 5.1. Descargar BusyBox

Descarga la última versión estable de BusyBox (por ejemplo, la versión 1.36.1):

```bash
cd ..
wget https://busybox.net/downloads/busybox-1.36.1.tar.bz2
```

### 5.2. Extraer BusyBox

```bash
tar xf busybox-1.36.1.tar.bz2
```

## Paso 6: Configurar y Compilar BusyBox

### 6.1. Entrar en el Directorio de BusyBox

```bash
cd busybox-1.36.1
```

### 6.2. Configurar BusyBox con la Configuración Predeterminada

```bash
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- defconfig
```

### 6.3. Configurar BusyBox para Enlace Estático

Ejecuta:

```bash
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- menuconfig
```

En el menú de configuración:

- Navega a **BusyBox Settings -> Build Options**.
- Marca la opción **Build BusyBox as a static binary (no shared libs)**.

Guarda y sal del menú.

### 6.4. Compilar BusyBox

```bash
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- -j$(nproc)
```

### 6.5. Instalar BusyBox en un Directorio Temporal

```bash
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- install
```

Esto instalará los archivos en `busybox-1.36.1/_install`.

## Paso 7: Crear el Sistema de Archivos Raíz

### 7.1. Crear el Directorio `rootfs`

Desde el directorio principal de tu proyecto:

```bash
cd ..
mkdir -p rootfs
```

### 7.2. Copiar los Archivos de BusyBox a `rootfs`

```bash
cp -r busybox-1.36.1/_install/* rootfs/
```

### 7.3. Crear el Script de Inicio `init`

Crea el archivo `init`:

```bash
sudo nano rootfs/init
```

Contenido del archivo `init`:

```sh
#!/bin/sh

mount -t proc none /proc
mount -t sysfs none /sys

echo -e "\nBooting minimal ARM Linux system...\n"

exec /bin/sh
```

Haz que el script sea ejecutable:

```bash
sudo chmod +x rootfs/init
```

### 7.4. Crear Directorios Necesarios

```bash
mkdir -p rootfs/{dev,proc,sys,usr,bin,sbin,etc}
```

## Paso 8: Crear el Archivo `initramfs.cpio.gz`

### 8.1. Empaquetar el Sistema de Archivos Raíz

Desde el directorio `rootfs`:

```bash
cd rootfs
find . | cpio -H newc -ov --owner root:root | gzip -9 > ../initramfs.cpio.gz
```

Esto crea el archivo `initramfs.cpio.gz` en el directorio principal.

## Paso 9: Ejecutar QEMU

### 9.1. Volver al Directorio Principal

```bash
cd ..
```

### 9.2. Ejecutar QEMU

```bash
qemu-system-arm -M versatilepb -kernel linux-6.5.7/arch/arm/boot/zImage \
-dtb linux-6.5.7/arch/arm/boot/dts/arm/versatile-pb.dtb \
-initrd initramfs.cpio.gz -append "console=ttyAMA0" -serial mon:stdio -nographic
```

**Explicación de las Opciones:**

- `-M versatilepb`: Emula la placa VersatilePB.
- `-kernel`: Ruta al kernel compilado (`zImage`).
- `-dtb`: Ruta al Device Tree Blob (`versatile-pb.dtb`).
- `-initrd`: Ruta al archivo `initramfs.cpio.gz`.
- `-append "console=ttyAMA0"`: Especifica la consola serial.
- `-serial mon:stdio`: Redirige la salida de la consola y el monitor de QEMU a tu terminal.
- `-nographic`: Ejecuta QEMU sin interfaz gráfica.

## Paso 10: Verificar el Arranque del Sistema

Si todo está configurado correctamente, deberías ver algo como:

```bash
Booting minimal ARM Linux system...

/ #
```

Esto indica que has iniciado sesión en tu sistema Linux minimalista en QEMU ARM.
