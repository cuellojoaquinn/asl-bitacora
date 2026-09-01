# P01 — Onboarding del laboratorio

## 1. Escenario y topología
Dos VMs Debian 12 clonadas de una plantilla base, conectadas por la red
interna lan-empresa (192.168.100.0/24). srv1 = .10 (servidor),
cli1 = .100 (estación de trabajo). Adaptador 1 en NAT solo para apt.
Laboratorio armado en UTM (Apple Silicon) en lugar de VirtualBox.

## 2. Procedimiento
- Instalación de Debian 12 en una VM base (sin entorno gráfico, con
  usuario sysadmin en el grupo sudo y contraseña de root vacía).
- Preflight de paquetes y contrato de la plantilla (openssh-server,
  git, man-db, vim, curl, tmux, htop, rsync, tcpdump) verificado OK.
- Snapshot `plantilla-limpia` sobre la VM base con la VM apagada.
- Clonado de la plantilla en dos VMs: srv1 y cli1.
- Corrección de hostname en cada clon (`hostnamectl set-hostname` +
  edición de /etc/hosts), ambos heredaban el nombre `base`.
- Identificación de la interfaz de red interna con `ip -br a` (en UTM
  se llama enp0s2, no enp0s8 como en el ejemplo de VirtualBox).
- Configuración de IP estática en la red interna lan-empresa:
  srv1 = 192.168.100.10/24, cli1 = 192.168.100.100/24, vía
  /etc/network/interfaces.d/lan-empresa.
- Verificación cruzada: ping, ssh y hostnamectl desde cli1 hacia srv1.
- Snapshot de cierre `pre-p02` en ambas VMs (apagadas), tomado con
  `qemu-img snapshot -c pre-p02 <disco>.qcow2` sobre cada archivo
  .qcow2 ubicado en
  ~/Library/Containers/com.utmapp.UTM/Data/Documents/<vm>.utm/Data/,
  ya que UTM no tiene gestión de snapshots por interfaz gráfica.
- Creación del repositorio Git de la bitácora (asl-bitacora) y clonado
  en cli1 mediante HTTPS con autenticación por Personal Access Token.

## 3. Evidencia de verificación

### Ping cli1 -> srv1
```
$ ping -c3 192.168.100.10
PING 192.168.100.10 (192.168.100.10) 56(84) bytes of data.
64 bytes from 192.168.100.10: icmp_seq=1 ttl=64 time=2.26 ms
64 bytes from 192.168.100.10: icmp_seq=2 ttl=64 time=0.590 ms
64 bytes from 192.168.100.10: icmp_seq=3 ttl=64 time=1.43 ms

--- 192.168.100.10 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2006ms
rtt min/avg/max/mdev = 0.590/1.424/2.258/0.680 ms
```

### SSH cli1 -> srv1
```
$ ssh sysadmin@192.168.100.10
sysadmin@192.168.100.10's password:
Last login: Tue Sep  1 18:24:00 2026
sysadmin@srv1:~$
```
El prompt cambia a sysadmin@srv1, confirmando acceso administrativo remoto.

### Identidad de srv1
```
$ hostnamectl | head -3
 Static hostname: srv1
       Icon name: computer-vm
         Chassis: vm
$ ip -br a
enp0s2  UP  192.168.100.10/24
```

### Identidad de cli1
```
$ hostnamectl | head -3
 Static hostname: cli1
       Icon name: computer-vm
         Chassis: vm
$ ip -br a
enp0s2  UP  192.168.100.100/24
```

## 4. Problemas encontrados

### 1. Snapshot duplicado en cli1
Al tomar el snapshot pre-p02 con `qemu-img snapshot -c`, el comando se
ejecutó dos veces por error, generando dos snapshots con el mismo tag
(ID 1 y 2). Diagnóstico: `qemu-img snapshot -l` mostró dos filas
idénticas. El intento de borrar por ID (`-d 2`) falló porque qemu-img
identifica snapshots por tag, no por ID; borrar por tag (`-d pre-p02`)
eliminó ambas entradas. Solución: se volvió a crear un único snapshot
pre-p02 limpio. No hubo pérdida de información real porque ambos
snapshots duplicados representaban el mismo estado de disco (VM
apagada, sin cambios entre uno y otro).

### 2. UTM no tiene snapshots nativos en la interfaz gráfica
La guía asume el flujo de VirtualBox (menú Instantáneas -> Tomar), que
no existe en UTM. Diagnóstico: investigación en la documentación y el
repositorio de UTM en GitHub, confirmando que la función de snapshots
por interfaz está pedida pero no implementada. Solución: se usó
`qemu-img snapshot` por línea de comandos directamente sobre los
archivos .qcow2 de cada VM, con la VM apagada en cada caso.

### 3. Direcciones MAC duplicadas tras el clonado
Al conectarme por SSH desde la Mac, la conexión se cortaba de forma
intermitente (Broken pipe) y ambas VMs resultaron tener la misma IP en
el adaptador NAT. Diagnóstico: `ip link show enp0s1` en cada VM mostró
la misma dirección MAC en las dos. A diferencia de VirtualBox, UTM no
regenera automáticamente las direcciones MAC al clonar una VM: srv1 y
cli1 heredaron la MAC de la plantilla base sin cambios. El DHCP de UTM
(vmnet de Apple, red 192.168.64.0/24) identifica clientes por MAC, no
por nombre de VM, y entregó la misma IP a ambas al no poder
distinguirlas. Solución: se cambió manualmente la dirección MAC del
adaptador NAT en cli1, se eliminó la IP vieja superpuesta y se renovó
la asignación DHCP para dejar una sola dirección limpia por VM.

### 4. Fallo de resolución DNS al clonar el repositorio Git
`git clone` sobre HTTPS falló con "Could not resolve host: github.com".
Diagnóstico: `ping -c3 8.8.8.8` confirmó conectividad IP, aislando el
problema a resolución de nombres; /etc/resolv.conf no tenía ninguna
línea nameserver. Es el mismo problema recurrente ya enfrentado en
prácticas anteriores del laboratorio (el adaptador NAT de UTM no
siempre entrega DNS por DHCP). Solución aplicada:
```
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```
Nota: esta solución no persiste ante un reinicio o renovación DHCP;
queda pendiente fijarla de forma permanente en la configuración de red.

### 5. Autenticación fallida contra GitHub
GitHub ya no acepta contraseña de cuenta para operaciones Git por
HTTPS (deprecado desde 2021). Solución: generación de un Personal
Access Token (classic) con scope repo desde GitHub, usado como
contraseña en el prompt de git clone.

## 5. Análisis
La mayoría de los problemas de esta práctica (snapshots por interfaz,
MACs no regeneradas al clonar, nombres de interfaz de red) no son
errores de configuración propios sino consecuencia de que la guía de
la cátedra está escrita asumiendo VirtualBox, mientras el laboratorio
se armó en UTM sobre Apple Silicon. Esto obligó a verificar cada
supuesto de la guía contra el comportamiento real del sistema en vez
de seguir los pasos de forma literal (por ejemplo, usar `ip -br a`
para confirmar el nombre de la interfaz en lugar de asumir enp0s8, o
recurrir a qemu-img en lugar de un menú de snapshots inexistente). El
problema de DNS, en cambio, es recurrente y de origen distinto: el
adaptador NAT de UTM no siempre entrega un DNS funcional por DHCP, lo
que sugiere que conviene resolverlo de forma permanente antes de P02
en lugar de repetir el parche manual en cada práctica.

## 6. Conclusión
El laboratorio quedó operativo con srv1 y cli1 comunicándose por la
red interna lan-empresa, cada una con identidad y direccionamiento
propios, verificado por ping, SSH y hostnamectl. Se estableció el
punto de retorno pre-p02 en ambas VMs y se dejó la bitácora Git
configurada en cli1 para el resto del semestre. Las diferencias entre
UTM y VirtualBox fueron el eje central de esta práctica y quedan
documentadas para no repetir el mismo diagnóstico en prácticas
futuras.
