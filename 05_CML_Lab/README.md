# Laboratorio CCNA: Cisco Modeling Labs Free en virt-manager (Linux)

Guía de instalación de **Cisco Modeling Labs (CML) Free 2.10.1** como máquina virtual en **virt-manager / libvirt (KVM)** sobre Linux (Omarchy, basado en Arch). Incluye los pasos que funcionaron, los errores que aparecieron y cómo se resolvieron.

> Estado: CML instalado y funcionando, con el refplat copiado y las imágenes de nodos disponibles. La integración con MCP queda pendiente (ver el último apartado).

---

## 1. Resumen rápido

| Elemento | Valor |
|---|---|
| Producto | Cisco Modeling Labs Free 2.10.1 (build 21) |
| Hipervisor | virt-manager / libvirt (KVM) |
| Nombre de la VM | `BW_Cisco_CML` (hostname `bw-cml`) |
| Firmware | **UEFI** (`OVMF_CODE.4m`, sin secboot) |
| CPU | 4 vCPUs, *Copy host CPU configuration* (host-passthrough) |
| RAM | 8 GB como mínimo; recomendado 16 GB si el equipo lo permite |
| Disco | 64 GB, qcow2, bus VirtIO |
| Red | NAT (`default`) |
| Interfaz web | `https://<IP-de-la-VM>/` |
| Consola de administración (Cockpit) | `https://<IP-de-la-VM>:9090/` |
| SSH | Puerto `1122` (no el 22) |
| Límite de la edición Free | Hasta 5 nodos en ejecución a la vez |

> **Por qué CML además de Packet Tracer:** Packet Tracer simula el comportamiento de los equipos; CML ejecuta imágenes reales de IOS (IOL, IOL-L2). Sirve para contrastar la salida de `show` y el comportamiento de protocolos (OSPF, STP, ACL) con un entorno más cercano al examen y al trabajo real. Los laboratorios de Packet Tracer del repositorio (`02_Packet_Tracer_Labs/`) se pueden replicar aquí para comparar.

---

## 2. Requisitos previos

- Procesador con **virtualización anidada** activada. Cisco soporta oficialmente CPUs Intel; en AMD puede funcionar, pero sin garantía.
- virt-manager y libvirt instalados.
- Unos **40 GB libres** para descargas y archivos descomprimidos, más el disco de la VM.
- Una cuenta Cisco (CCO ID) con el perfil completo.

Comprobar la virtualización anidada en Intel:

```bash
cat /sys/module/kvm_intel/parameters/nested
```

Debe devolver `Y` o `1`. En AMD:

```bash
cat /sys/module/kvm_amd/parameters/nested
```

---

## 3. Descarga de los archivos

### 3.1 Cuenta Cisco

1. Crear la cuenta en <https://id.cisco.com/>.
2. Registrarse en <https://mkto.cisco.com/cml-free.html>.
3. Completar **todo** el perfil. Si no se hace, la descarga muestra un aviso pidiendo la dirección completa (restricciones de exportación de EE. UU.). Si no se trabaja para una empresa, se puede indicar `Self` como empresa.

### 3.2 Archivos correctos (sección "Free")

En Software Central hay que entrar en la sección **Free**, no en la release 2.10.0 de pago. Los archivos de la edición de pago (refplat de 14 GB, suplementario, ISE) **no** son los de CML Free.

Archivos necesarios:

| Archivo | Tamaño aprox. | Uso |
|---|---|---|
| `cml2_f_2.10.1-21_amd64-21-iso.zip` | 4,1 GB | Instalador del servidor (bare metal), el que se usa en virt-manager |
| `refplat-20260409-free-iso.zip` | 3 GB | Imágenes de nodos de CML Free (obligatorio si no se migra desde otra instalación) |

Archivos que **no** hacen falta:

- `cml2_f_2.10.1-21_amd64-21.ova`: solo para VMware.
- `cml2_f_2.10.1-21_amd64-23-pkg.zip`: solo para actualizar un servidor ya instalado.

### 3.3 Descomprimir

Cada zip en su propia carpeta, para no mezclar los archivos que contienen:

```bash
cd /ruta/a/Cisco_CML
mkdir -p cml-servidor cml-refplat
unzip cml2_f_2.10.1-21_amd64-21-iso.zip -d cml-servidor
unzip refplat-20260409-free-iso.zip -d cml-refplat
ls -lh cml-servidor cml-refplat
```

Resultado esperado:

- `cml-servidor/cml2_2.10.1-21_amd64-21.iso` (≈ 4,4 GB)
- `cml-refplat/refplat-20260409-free.iso` (≈ 3,6 GB)

Junto a cada ISO aparecen también un `.pem`, un `.signature`, un `.README` y el script `cisco_x509_verify_release.py3` para verificar la firma. Son opcionales.

### 3.4 Copiar los ISO a la carpeta de libvirt

Desde un disco externo, libvirt puede dar *permission denied*. Para evitarlo se copian a su carpeta:

```bash
df -h /var/lib/libvirt/images
sudo cp cml-servidor/cml2_2.10.1-21_amd64-21.iso cml-refplat/refplat-20260409-free.iso /var/lib/libvirt/images/
ls -lh /var/lib/libvirt/images/
```

---

## 4. Crear la VM en virt-manager

1. **Nueva VM** → *Local install media (ISO image or CDROM)*.
2. Seleccionar `/var/lib/libvirt/images/cml2_2.10.1-21_amd64-21.iso`.
3. Sistema operativo: `ubuntu 26` (es solo una etiqueta para valores por defecto).
4. Memoria y CPU: virt-manager propone 6144 MB y 2 CPUs, que se quedan cortos. Poner **16384 MB** (mínimo 8192) y **4 CPUs**.
5. Disco: *Create a disk image for the virtual machine*, **64 GB**.
6. Nombre `BW_Cisco_CML`, red NAT y marcar **Customize configuration before install**.
7. En la ventana de personalización:
   - **Overview → Firmware → UEFI x86_64** (la opción `code 4m`, `OVMF_CODE.4m.fd`, **sin** `secboot`). Pulsar *Apply*.
   - **CPUs → Copy host CPU configuration** marcado (suele venir por defecto).
   - **Add Hardware → Storage → Select or create custom storage** → elegir `refplat-20260409-free.iso` con *Device type* **CDROM device**.
8. *Begin Installation*.

> **Importante:** el firmware solo se puede cambiar antes de empezar la instalación. En una VM ya creada el campo aparece en gris, y hay que borrarla y recrearla.

---

## 5. Asistente de instalación

Tras el arranque aparece el instalador de Ubuntu (subiquity) y después el asistente **CML^2 Initial System Configuration**:

- **Hostname**: un nombre en minúsculas, sin espacios (por ejemplo `bw-cml`).
- **Cuentas**: el asistente pide dos cuentas distintas (ver apartado 7).
- **IPv4**: dejar **DHCP**; la red NAT de virt-manager reparte la IP.
- **MTU**: dejar **1500**.
- **Servicios opcionales**:
  - **OpenSSH**: SSH en el puerto 1122. Activado.
  - **MCPServer**: servidor MCP para LLMs. Activado.
  - **PATty**: reenvío de puertos de los nodos del laboratorio. No activado.
- **Reference Platform**: el instalador detecta el ISO del refplat conectado como segundo CD y lo copia al disco de la VM (unos 10 minutos). Después se puede quitar ese CD de la VM.

Al terminar, la consola muestra las direcciones de la interfaz web y de Cockpit.

---

## 6. Primer acceso

- Interfaz de CML: `https://<IP-de-la-VM>/`
- Cockpit: `https://<IP-de-la-VM>:9090/`

El certificado es autofirmado, así que el navegador avisa. Hay que aceptar el riesgo para continuar.

Para comprobar que el refplat se copió bien, en la web de CML: **Tools → Node and Image Definitions → Image Definitions**. Debe aparecer una lista de imágenes (IOL, IOL-L2, ASAv, Alpine, Ubuntu, etc.).

Si hace falta recopiar el refplat más adelante, Cockpit tiene la acción **CML2 → Copy Refplat ISO**, con el ISO conectado a la VM.

---

## 6 bis. Uso diario de la VM

Comandos de libvirt para manejar la VM sin abrir virt-manager (nombre de la VM: `BW_Cisco_CML`):

```bash
virsh --connect qemu:///system list --all                      # estado
virsh --connect qemu:///system start BW_Cisco_CML              # arrancar
virsh --connect qemu:///system shutdown BW_Cisco_CML           # apagado limpio (ACPI)
virsh --connect qemu:///system snapshot-create-as BW_Cisco_CML base-limpia   # instantánea
virsh --connect qemu:///system snapshot-list BW_Cisco_CML
```

Buenas prácticas:

- **Apagar siempre con `shutdown`** (o desde Cockpit) y no con *Force Off*; un corte brusco puede corromper el disco.
- **Hacer una instantánea justo después de instalar** y antes de tocar nada (`base-limpia`). Es la forma más barata de evitar un factory reset.
- **Exportar cada topología** (descargarla como YAML desde la interfaz web) y guardarla en `topologias/`: las instantáneas protegen la VM, el YAML protege el laboratorio.
- Con el límite de 5 nodos, planificar topologías pequeñas: 2 routers + 2 switches + 1 host es suficiente para casi todo el temario (VLANs, trunking, rutas estáticas, OSPF, ACL).

---

## 7. Cuentas y credenciales

CML usa **dos cuentas separadas**:

| Cuenta | Dónde sirve |
|---|---|
| Usuario del **sistema** (sysadmin) | Consola de la VM, SSH y **Cockpit** (`:9090`) |
| Usuario de la **interfaz web** (admin de CML) | Web de CML, API y consola de nodos |

No son intercambiables. Si falla el acceso a la web con la cuenta del sistema, es normal.

> **Seguridad:** no subir nunca contraseñas, IPs de acceso ni capturas con credenciales a este repositorio (es público). Usar un gestor de contraseñas. Si tras un factory reset se usa el contenido de `/etc/machine-id` como contraseña inicial, cambiarla de inmediato.

### Si se pierde la contraseña de la web de CML

Según la FAQ de Cisco:

- Si hay otro administrador, puede restablecerla en **Tools → System Administration → User Administration**.
- Si se es el único administrador, la única vía documentada es un **Factory Reset**, que es **destructivo**: borra usuarios, laboratorios, definiciones de nodos e imágenes. Se hace en **Cockpit → CML2 → Factory Reset**. Después se crea el usuario `cml2` con la contraseña igual al contenido de `/etc/machine-id`, y la contraseña se cambia en la web.
- El script `virl2-switch-admin.sh` está descrito como la forma de reemplazar la cuenta de fábrica, pero intenta cambiar la cuenta `cml2` con su contraseña por defecto. Si esa cuenta ya no existe, la API devuelve `401 Unauthorized`.

Tras un factory reset hay que volver a copiar el refplat.

---

## 8. Problemas encontrados y soluciones

| Problema | Causa | Solución |
|---|---|---|
| `autoinstall config did not create needed bootloader partition` | La VM se creó con BIOS; el instalador necesita UEFI | Recrear la VM con firmware **UEFI x86_64** (`OVMF_CODE.4m`) |
| El campo *Firmware* sale en gris | Solo se puede cambiar antes de instalar | Borrar la VM y recrearla; al borrar, desmarcar los ISO y dejar marcado solo el disco `.qcow2` |
| Aviso de perfil incompleto al descargar | Perfil Cisco sin la dirección completa | Completar el perfil en <https://id.cisco.com/>; probar ventana de incógnito si persiste |
| Archivos de descarga equivocados | Se entró en la release 2.10.0 de pago, no en **Free** | Usar la sección **Free** de Software Central |
| Teclado distinto dentro de la consola de la VM | La consola de la VM usa distribución inglesa | Usar la terminal de Cockpit en el navegador, que usa el teclado del equipo |
| `unzip` mezcla archivos sueltos | Se descomprimió sin `-d` | Descomprimir cada zip en su carpeta con `-d` |
| Autenticación fallida en la web de CML | Se usó la cuenta del sistema o una contraseña distinta | Ver apartado 7 |

---

## 9. Pendiente: integración con MCP

- CML 2.10 ofrece un servicio **MCPServer** que se activa en el asistente.
- No se ha encontrado documentación oficial de Cisco sobre cómo conectarlo a un cliente.
- Opciones valoradas:
  - **Claude Code** en la máquina local: llega a la IP privada de la VM sin exponer nada, pero requiere un plan de pago o una cuenta de API.
  - **Conector desde claude.ai**: requeriría una URL pública con HTTPS, lo que expone el controlador a internet y no se recomienda.
  - **Sin MCP**: usar la interfaz web de CML y pedir las topologías y configuraciones al asistente. Es la opción elegida por ahora.

---

## 10. Siguientes pasos del laboratorio

- Crear un primer laboratorio con dos routers IOL conectados.
- Practicar direccionamiento IP, rutas estáticas, VLANs y OSPF según avance el temario CCNA, priorizando *IP Connectivity* (25 %) y *Network Access* (20 %).
- Guardar las topologías (YAML) y las configuraciones en `topologias/`, un subdirectorio por laboratorio, con su propio `README.md` (objetivo, topología, configuración, verificación con `show`, fallos encontrados).
- Seguir la regla del repositorio: intentar el laboratorio, fallar, analizar el `show` y corregir; no copiar y pegar configuraciones.

| # | Laboratorio CML | Tema | Estado |
|---|---|---|---|
| C01 | 2 routers IOL, direccionamiento y ruta estática | IP Connectivity | Pendiente |
| C02 | VLANs y trunking con IOL-L2 | Network Access | Pendiente |
| C03 | OSPFv2 de área única | IP Connectivity | Pendiente |

---

## Qué se sube a este repositorio (y qué no)

- **Sí:** esta guía, topologías exportadas en YAML, configuraciones y notas de verificación.
- **No:** los ISO (`cml2_*.iso`, `refplat-*.iso`), los zips de Cisco, los discos `.qcow2`, contraseñas ni capturas con datos de acceso. El `.gitignore` de la raíz los excluye; además, el software de Cisco tiene licencia propia y no debe redistribuirse.

---

## Referencias

- Documentación de CML: <https://developer.cisco.com/docs/modeling-labs/>
- CML Free: <https://developer.cisco.com/docs/modeling-labs/cml-free>
- Descargas (Software Central, sección CML-Free): <https://software.cisco.com/download/home/286193282/type/286326381/release/CML-Free>
