---
title: "HappyCorp - Entorno de AD con misconfiguraciones desde Cero"
date: 2026-07-31 10:00:00 +0100
categories: [proyectos]
tags: [active-directory, homelab, crtp, redteam, windows]
pin: true
---

<pre style="background: none; border: none; box-shadow: none; padding: 0; text-align: center; font-size: 1rem; line-height: 1.2;">
.-""""""-.
/  ^    ^  \
|   ⌣  ⌣    |
 \  \____/  /
  '-......-'

    __  _____    ____  ______  __   __________  ____  ____ 
   / / / /   |  / __ \/ __ \ \/ /  / ____/ __ \/ __ \/ __ \
  / /_/ / /| | / /_/ / /_/ /\  /  / /   / / / / /_/ / /_/ /
 / __  / ___ |/ ____/ ____/ / /  / /___/ /_/ / _, _/ ____/ 
/_/ /_/_/  |_/_/   /_/     /_/   \____/\____/_/ |_/_/      
</pre>

## Introducción

Esta documentación explica cómo configurar, una a una, un conjunto de vulnerabilidades y misconfiguraciones típicas de un entorno de Active Directory. No cubre cómo explotarlas ni qué herramientas usar para ello — el objetivo no es un writeup de ataque, sino una guía de construcción.

La idea es poder aplicar cada misconfiguration de forma aislada, entender exactamente qué la provoca y por qué es peligrosa, y usarla después como base para practicar tu propia metodología de enumeración y explotación. De paso, se gana un conocimiento mucho más profundo de lo que ocurre "por debajo" en un entorno de directorio activo, más allá de simplemente lanzar una herramienta y ver que funciona.

## Objetivo del laboratorio

Esto **no** es un entorno ya montado con una ruta de ataque predefinida de principio a fin, al estilo de proyectos como GOAD. No es ese el enfoque aquí.

El objetivo es que puedas levantar tu propio entorno, entender cada pieza que lo compone, y decidir tú mismo qué misconfiguraciones introducir y cómo encajan entre sí — en lugar de heredar un escenario ya diseñado por otra persona.

Eres libre de modificar los comandos a tu elección para crear tu propio path de ataque.

## Arquitectura de red

![Topología de HappyCorp](/assets/img/happycorp.png)

No es necesario tener todas las máquinas encendidas a la vez — es un consumo de recursos considerable, y para trabajar sobre una misconfiguration concreta normalmente basta con el DC correspondiente y una o dos máquinas más. Además, esta topología no es un límite: se puede ampliar con más forests, más dominios hijo, o más equipos según lo que quieras practicar.

## Fase 1 — Despliegue de las máquinas virtuales

Máquinas a desplegar, según la topología:

| Nombre | Rol | Forest / Dominio | IP |
|---|---|---|---|
| `happy-dc` | Domain Controller (forest root) | `happy.corp` | 192.168.101.10 |
| `us-happy-dc` | Domain Controller (child domain) | `us.happy.corp` | 192.168.101.11 |
| `happy-mgmt` | Servidor de gestión | `happy.corp` | 192.168.101.12 |
| `happy-sql` | SQL Server | `happy.corp` | 192.168.101.13 |
| `happy-websvc` | Servidor de aplicaciones / Jenkins | `happy.corp` | 192.168.101.14 |
| `us-happy-sql` | SQL Server | `us.happy.corp` | 192.168.101.15 |
| `happy-n3oari` | Windows 10 — equipo del atacante (assumed breach) | `happy.corp` | 192.168.101.50 |

Todas las máquinas comparten la misma red **host-only** de VMware (subred `192.168.101.0/24`, sin DHCP) — crea una interfaz host-only y conecta todas las VMs a ella. La misma subred vale tanto para `happy.corp` como para `us.happy.corp`, no hace falta una red separada por dominio.

Para cada VM: instalar el sistema operativo, renombrar el equipo, y configurar IP estática (sin gateway, ya que es una red aislada sin salida a internet) apuntando como DNS primario al DC correspondiente de su dominio (`happy-dc` para las máquinas de `happy.corp`, `us-happy-dc` para las de `us.happy.corp`).

Como recomendación, para ahorrar recursos conviene instalar las máquinas sin interfaz gráfica (Server Core). Esto se configura por PowerShell:

```powershell
Rename-Computer -NewName "happy-mgmt" -Restart

# Tras el reinicio, IP estática (ajustando el nombre de la interfaz con Get-NetAdapter si hace falta)
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.101.12 -PrefixLength 24
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 192.168.101.10
```

## Fase 2 — Creación de los forests y dominios

**`happy-dc`** — se promociona como el primer controlador de dominio, creando un nuevo forest:

```markdown
Add roles and features → Active Directory Domain Services → Promote this server to a domain controller
→ Add a new forest → Root domain name: happy.corp
```

> En "DNS Options" aparecerá un aviso de que no se puede crear una delegación DNS — es normal en un forest nuevo sin dominio padre, se ignora sin problema.
{: .prompt-tip }

**`us-happy-dc`** — se une al forest ya existente como dominio hijo:

```markdown
Add roles and features → Active Directory Domain Services → Promote this server to a domain controller
→ Add a new domain to an existing forest → Child Domain
→ Parent domain: happy.corp | New domain: us
→ Credenciales: HAPPY\Administrator
```

El resto de equipos (`happy-mgmt`, `happy-sql`, `happy-websvc`, `us-happy-sql`) se unen a su dominio correspondiente como miembros normales:

```powershell
Add-Computer -DomainName "happy.corp" -Credential "happy\Administrator" -Restart
```

## Fase 3 — Establecimiento del trust bidireccional

Al crear `us.happy.corp` como dominio hijo de `happy.corp`, Active Directory establece automáticamente un trust padre-hijo transitivo y bidireccional (`WITHIN_FOREST`) — no requiere configuración manual adicional.

Verificación, desde cualquier equipo del dominio con el módulo `ActiveDirectory` instalado:

```powershell
Get-ADTrust -Filter *
# happy.corp -> us.happy.corp
# Transitivo: True | Dirección: Bidirectional
```

## Preparar el punto de entrada (assumed breach)

Antes de introducir misconfiguraciones, creamos `n3oari`: el usuario de dominio con privilegios mínimos que representa el punto de entrada del atacante — el típico "assumed breach" del que se parte en un ejercicio de red team, donde ya se asume una credencial de bajo privilegio comprometida.

```powershell
New-ADUser -Name "n3oari" -SamAccountName "n3oari" -UserPrincipalName "n3oari@happy.corp" -AccountPassword (ConvertTo-SecureString "Password123!" -AsPlainText -Force) -Enabled $true -Path "CN=Users,DC=happy,DC=corp"
```

Para poder operar cómodamente desde esa cuenta durante el laboratorio, habilitamos acceso remoto en `happy-n3oari`:

```powershell
# WinRM
Enable-PSRemoting -Force
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "*" -Force
netsh advfirewall firewall add rule name="WinRM" protocol=TCP dir=in localport=5985 action=allow

# RDP
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"

# Permitir ping, útil para comprobar conectividad entre VMs
netsh advfirewall firewall add rule name="Allow ICMP" protocol=icmpv4:8,any dir=in action=allow

# Añadir n3oari a los grupos locales necesarios
Add-LocalGroupMember -Group "Remote Management Users" -Member "happy\n3oari"
Add-LocalGroupMember -Group "Remote Desktop Users" -Member "happy\n3oari"
```

> Para que las herramientas ofensivas que uses más adelante no se vean interferidas durante la práctica, es habitual desactivar Windows Defender en las máquinas del laboratorio. No es una misconfiguration a documentar, es simplemente higiene de laboratorio — en un entorno real jamás se haría esto.
> ```powershell
> Set-MpPreference -DisableRealtimeMonitoring $true
> Set-MpPreference -DisableIOAVProtection $true
> Set-MpPreference -DisableBehaviorMonitoring $true
> Set-ExecutionPolicy Unrestricted -Force
> reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows Defender" /v DisableAntiSpyware /t REG_DWORD /d 1 /f
> ```
{: .prompt-tip }

## Fase 4 — Introducción de misconfiguraciones

Misconfiguraciones cubiertas en esta documentación:

- [x] Recursos compartidos permisivos / exposición de información
- [x] Escalada de privilegios por Unquoted Service Path
- [x] Abuso de ACLs
- [x] Abuso de GPOs
  - [x] Añadir administrador local vía GPO
  - [x] Deshabilitar/debilitar la GPO de AppLocker
- [x] Abuso de Jenkins mal configurado
- [x] Kerberoasting
- [x] AS-REP Roasting
- [x] Delegaciones
  - [x] Unconstrained Delegation
  - [x] Constrained Delegation
  - [x] Resource-Based Constrained Delegation (RBCD)
- [x] SQL Server mal integrado en el dominio
- [x] Plantillas de certificado (ADCS) vulnerables

### Recursos compartidos con información sensible

Uno de los hallazgos más comunes en una auditoría interna de Active Directory no es una vulnerabilidad técnica sofisticada, sino simple negligencia operativa: carpetas compartidas por SMB con permisos demasiado abiertos, que terminan exponiendo credenciales, scripts de administración o configuraciones internas a cualquier usuario del dominio (o a todos, sin autenticar).

Vamos a replicar tres escenarios típicos, cada uno con un nivel de exposición distinto, para tener variedad a la hora de practicar enumeración.

**Escenario 1 — Script de backup con credenciales en texto plano**

Un administrador automatiza un backup y dentro del script deja embebida la contraseña de una cuenta de servicio. El share se comparte con "Domain Users" en modo lectura porque "total, solo lo necesitan ver los del equipo de sistemas".

```powershell
mkdir C:\Scripts
Set-Content -Path "C:\Scripts\backup.ps1" -Value '$cred = New-Object PSCredential("happy\svc_bckjob", (ConvertTo-SecureString "Password123!" -AsPlainText -Force))'
icacls "C:\Scripts" /grant "happy\Domain Users:(OI)(CI)R" /T
New-SmbShare -Name "Scripts" -Path "C:\Scripts" -ReadAccess "happy\Domain Users"
```

**Escenario 2 — Configuración de aplicación con permisos de "Everyone"**

Un share de IT con un archivo de configuración de una aplicación interna, expuesto con control total para "Everyone" (incluye acceso anónimo/no autenticado en función de la política de la red). Es el tipo de hallazgo que suele aparecer cuando alguien monta un recurso "temporal" para pruebas y se olvida de retirarlo.

```powershell
mkdir C:\IT
Set-Content -Path "C:\IT\app.config" -Value '<configuration><appSettings><add key="svcAccount" value="happy\svc_webapp"/><add key="svcPassword" value="Password123!"/></appSettings></configuration>'
icacls "C:\IT" /grant "Everyone:(OI)(CI)F" /T
New-SmbShare -Name "IT" -Path "C:\IT" -FullAccess "Everyone"
```

**Escenario 3 — Notas internas del equipo de gestión, en un servidor de administración**

Un tercer share, esta vez en `happy-mgmt`, simula notas internas de un equipo de operaciones — el tipo de documento que revela nombres de servidores críticos, procesos internos o incluso credenciales sueltas dejadas "temporalmente".

```powershell
Invoke-Command -ComputerName happy-mgmt -ScriptBlock {
    mkdir C:\Management
    Set-Content -Path "C:\Management\notas-internas.txt" -Value 'Recordatorio: cambiar la contraseña de svc_bckjob antes de fin de mes. Sigue siendo Password123! desde que se creó la cuenta.'
    icacls "C:\Management" /grant "happy\Domain Users:(OI)(CI)R" /T
    New-SmbShare -Name "Management" -Path "C:\Management" -ReadAccess "happy\Domain Users"
}
```

> Los tres escenarios comparten un patrón real: la información sensible no está en la carpeta por accidente técnico, sino porque alguien priorizó la comodidad operativa sobre el principio de mínimo privilegio. Es exactamente lo que un red teamer busca en las primeras fases de enumeración post-compromiso.
{: .prompt-tip }

### Servicio con ruta sin comillas (Unquoted Service Path)

Cuando la ruta del binario de un servicio de Windows contiene espacios y no está entre comillas, el sistema la interpreta probando cada segmento separado por espacios como un ejecutable válido, de izquierda a derecha. Si un usuario con pocos privilegios puede escribir en alguna de esas rutas intermedias, puede colocar ahí un binario que el sistema ejecutará con los privilegios del servicio — normalmente `SYSTEM`.

Lo replicamos en `happy-n3oari`, el equipo desde el que arranca el escenario de assumed breach:

```powershell
# Directorio y binario "real" del servicio
New-Item -ItemType Directory -Path "C:\Program Files\Vulnerable Service" -Force
New-Item -ItemType File -Path "C:\Program Files\Vulnerable Service\svc.exe" -Force

# Servicio creado SIN comillas en el binPath, corriendo como SYSTEM
sc.exe create "VulnSvc" binPath= "C:\Program Files\Vulnerable Service\svc.exe" start= auto obj= LocalSystem

# Para comparar: así se crearía de forma NO vulnerable, con el binPath entre comillas
# sc.exe create "VulnSvc" binPath= "\"C:\Program Files\Vulnerable Service\svc.exe\"" start= auto obj= LocalSystem

# Permisos de servicio abiertos: cualquiera puede pararlo/reiniciarlo
sc.exe sdset VulnSvc "D:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;WD)(A;;CCLCSWRPWPDTLOCRRC;;;SY)"

# Escritura habilitada para el usuario que representa el punto de entrada del atacante (assumed breach)
icacls "C:\Program Files" /grant "happy\n3oari:(OI)(CI)W" /T
```

Con esto queda expuesta la ruta intermedia `C:\Program Files\Vulnerable.exe` como punto de escritura para `n3oari`, el usuario que representa el foothold inicial del escenario (assumed breach) y desde el que se iría escalando privilegios.

### Abuso de ACLs y grupos

Las ACLs mal configuradas son, junto con las delegaciones, la fuente de rutas de escalada más habitual en cualquier entorno de AD real. Concedemos a `n3oari` distintos permisos sobre objetos concretos — cada uno abre una vía de escalada diferente.

Usuarios objetivo, uno por escenario:

```powershell
$pass = ConvertTo-SecureString "Password123!" -AsPlainText -Force
"m.garcia","r.silva","a.torres","l.fernandez","d.navarro" | ForEach-Object {
    New-ADUser -Name $_ -SamAccountName $_ -AccountPassword $pass -Enabled $true
}

$sidN3oari = (Get-ADUser n3oari).SID
```

**Escenario 1 — `GenericAll` sobre `m.garcia`** (control total: reset de contraseña, SPN, etc.)
```powershell
$acl = Get-Acl "AD:\CN=m.garcia,CN=Users,DC=happy,DC=corp"
$acl.AddAccessRule((New-Object System.DirectoryServices.ActiveDirectoryAccessRule($sidN3oari,"GenericAll","Allow")))
Set-Acl "AD:\CN=m.garcia,CN=Users,DC=happy,DC=corp" $acl
```

**Escenario 2 — `GenericWrite` sobre `r.silva`** (modificar atributos: preauth, SPN, script de logon)
```powershell
$acl = Get-Acl "AD:\CN=r.silva,CN=Users,DC=happy,DC=corp"
$acl.AddAccessRule((New-Object System.DirectoryServices.ActiveDirectoryAccessRule($sidN3oari,"GenericWrite","Allow")))
Set-Acl "AD:\CN=r.silva,CN=Users,DC=happy,DC=corp" $acl
```

**Escenario 3 — `WriteDACL` sobre el grupo `ServerAdmins`** (permite auto-concederse permisos sobre el grupo)
```powershell
$acl = Get-Acl "AD:\CN=ServerAdmins,CN=Users,DC=happy,DC=corp"
$acl.AddAccessRule((New-Object System.DirectoryServices.ActiveDirectoryAccessRule($sidN3oari,"WriteDacl","Allow")))
Set-Acl "AD:\CN=ServerAdmins,CN=Users,DC=happy,DC=corp" $acl
```

**Escenario 4 — `GenericAll` sobre el objeto equipo `happy-websvc`** (vía libre a RBCD sobre ese servidor)
```powershell
$acl = Get-Acl "AD:\CN=HAPPY-WEBSVC,CN=Computers,DC=happy,DC=corp"
$acl.AddAccessRule((New-Object System.DirectoryServices.ActiveDirectoryAccessRule($sidN3oari,"GenericAll","Allow")))
Set-Acl "AD:\CN=HAPPY-WEBSVC,CN=Computers,DC=happy,DC=corp" $acl
```

**Escenario 5 — `WriteDACL` sobre el objeto dominio** (permite auto-concederse derechos de DCSync)
```powershell
$acl = Get-Acl "AD:\DC=happy,DC=corp"
$acl.AddAccessRule((New-Object System.DirectoryServices.ActiveDirectoryAccessRule($sidN3oari,"WriteDacl","Allow")))
Set-Acl "AD:\DC=happy,DC=corp" $acl
```

**Cuenta de Domain Admin y grupos personalizados** (para tener objetivos "de valor" en el dominio):
```powershell
New-ADUser -Name "svc_dcadmin" -SamAccountName "svc_dcadmin" -AccountPassword $pass -Enabled $true
Add-ADGroupMember -Identity "Domain Admins" -Members "svc_dcadmin"

New-ADGroup -Name "ServerAdmins" -GroupScope Global
New-ADGroup -Name "RDPUsers" -GroupScope Global
```

### Abuso de GPOs

Igual que con las ACLs sobre objetos de usuario o equipo, una GPO mal protegida es una ruta directa a ejecución de código en todos los equipos donde se aplica. Creamos dos OUs, movemos equipos dentro y enlazamos GPOs con permisos débiles.

```powershell
# OUs
New-ADOrganizationalUnit -Name "DevOps" -Path "DC=happy,DC=corp"
New-ADOrganizationalUnit -Name "Servers" -Path "DC=happy,DC=corp"

# Mover equipos a sus OUs
Get-ADComputer HAPPY-MGMT | Move-ADObject -TargetPath "OU=Servers,DC=happy,DC=corp"
Get-ADComputer HAPPY-WEBSVC | Move-ADObject -TargetPath "OU=DevOps,DC=happy,DC=corp"

# GPOs y su enlace
New-GPO -Name "DevOps Policy" | New-GPLink -Target "OU=DevOps,DC=happy,DC=corp"
New-GPO -Name "Applocker Policy" | New-GPLink -Target "OU=Servers,DC=happy,DC=corp"
New-GPO -Name "ServerConfig" | New-GPLink -Target "OU=Servers,DC=happy,DC=corp"
```

**Escenario 1 — `WriteDACL` sobre "DevOps Policy"** (permite modificar la GPO y añadir un administrador local en todos los equipos de esa OU)
```powershell
$gpo = Get-GPO -Name "DevOps Policy"
$gpoPath = "AD:\CN={$($gpo.Id)},CN=Policies,CN=System,DC=happy,DC=corp"
$acl = Get-Acl $gpoPath
$acl.AddAccessRule((New-Object System.DirectoryServices.ActiveDirectoryAccessRule($sidN3oari,"WriteDacl","Allow")))
Set-Acl $gpoPath $acl
```

**Escenario 2 — `GenericAll` sobre "Applocker Policy"** (permite editar o retirar la política de AppLocker que se aplica a los servidores)
```powershell
$gpo = Get-GPO -Name "Applocker Policy"
$gpoPath = "AD:\CN={$($gpo.Id)},CN=Policies,CN=System,DC=happy,DC=corp"
$acl = Get-Acl $gpoPath
$acl.AddAccessRule((New-Object System.DirectoryServices.ActiveDirectoryAccessRule($sidN3oari,"GenericAll","Allow")))
Set-Acl $gpoPath $acl
```

Configuración base de la política de AppLocker (permisiva a propósito, para poder abusarla más adelante):

```powershell
# Activar el servicio necesario en el equipo donde se aplica la GPO
Start-Service AppIDSvc
```

```markdown
gpmc.msc → Forest → Domains → happy.corp → Group Policy Objects → Applocker Policy → Edit
Computer Configuration → Policies → Windows Settings → Security Settings →
Application Control Policies → AppLocker

- Executable Rules: "Everyone" permitido a ejecutar binarios firmados por Microsoft
- Script Rules: "Everyone" permitido a ejecutar scripts firmados por Microsoft desde cualquier ubicación
```

### Kerberoasting y AS-REP Roasting

Dos de las técnicas más rentables contra AD porque no requieren ningún privilegio previo: solo la capacidad de solicitar tickets Kerberos, algo que cualquier usuario autenticado del dominio puede hacer por diseño.

**Cuenta de servicio con SPN (Kerberoasting)** — cualquier cuenta con un Service Principal Name registrado es susceptible, porque su ticket de servicio va cifrado con el hash de la contraseña de esa cuenta.
```powershell
$pass = ConvertTo-SecureString "Password123!" -AsPlainText -Force
New-ADUser -Name "svc_sqlsvc" -SamAccountName "svc_sqlsvc" -AccountPassword $pass -Enabled $true
Set-ADUser -Identity "svc_sqlsvc" -ServicePrincipalNames @{Add='MSSQLSvc/happy-sql.happy.corp:1433'}
```

**Cuenta sin preautenticación Kerberos (AS-REP Roasting)** — si la preautenticación está deshabilitada, cualquiera puede pedir un AS-REP para esa cuenta sin necesidad de conocer su contraseña.
```powershell
New-ADUser -Name "vpn_remote" -SamAccountName "vpn_remote" -AccountPassword $pass -Enabled $true
Set-ADAccountControl -Identity vpn_remote -DoesNotRequirePreAuth $true
```

**Kerberoasting dirigido vía abuso de ACL** — no requiere configuración adicional: como `n3oari` ya tiene `GenericWrite` sobre `r.silva` (ver el escenario 2 de abuso de ACLs), esa cuenta queda igualmente expuesta a un Kerberoasting dirigido, añadiéndole un SPN al vuelo.

### Abuso de Jenkins mal configurado

Los servicios de terceros desplegados dentro del dominio, con cuentas de servicio propias y configuraciones por defecto, son otra vía de entrada habitual — no todo el abuso de AD pasa por objetos nativos de directorio.

Desplegamos Jenkins en `happy-websvc`, usando una cuenta de servicio del dominio para ejecutarlo:

```powershell
New-ADUser -Name "svc_jenkins" -SamAccountName "svc_jenkins" -AccountPassword (ConvertTo-SecureString "Password123!" -AsPlainText -Force) -Enabled $true
```

```markdown
Requisitos previos en happy-websvc:
- Java JDK 21 (https://adoptium.net/temurin/releases/?version=21)
- Jenkins LTS (https://www.jenkins.io/download/)

Antes de instalar, otorgar el privilegio "Log on as a service":
secpol.msc → Local Policies → User Rights Assignment → Log on as a service → Add → happy\svc_jenkins

Durante la instalación:
- Service Logon: happy\svc_jenkins / Password123!
- Plugins: "Select plugins to install" → desmarcar todos (instalación mínima)
- Usuario administrador de Jenkins: admin / admin   ← credenciales por defecto sin cambiar, la misconfiguration principal de este escenario

Jenkins escucha por defecto en el puerto 8080. El firewall de Windows lo bloquea para tráfico entrante, así que hay que abrir el puerto para poder acceder al panel desde otras máquinas del dominio:
netsh advfirewall firewall add rule name="Jenkins HTTP" dir=in action=allow protocol=TCP localport=8080
```

Con acceso al panel de administración vía esas credenciales por defecto, la superficie de ataque queda abierta: Jenkins permite definir "build steps" que se ejecutan con los privilegios de la cuenta de servicio (`svc_jenkins`), y su consola de scripts Groovy ofrece ejecución de código arbitrario directamente sobre el sistema operativo — un vector de RCE clásico una vez dentro del panel.

### Delegaciones

Las delegaciones Kerberos permiten que un servicio actúe en nombre de un usuario frente a otro servicio. Mal configuradas, son una de las rutas de escalada y movimiento lateral más potentes en cualquier entorno de AD.

**Unconstrained Delegation** — el equipo recibe el TGT completo de cualquier usuario que se autentique contra él, y puede reutilizarlo para suplantarlo frente a cualquier otro servicio del dominio.
```powershell
Set-ADComputer -Identity "happy-mgmt" -TrustedForDelegation $true
```

**Constrained Delegation** — la cuenta de servicio solo puede delegar hacia los servicios explícitamente listados en `msDS-AllowedToDelegateTo`.
```powershell
$pass = ConvertTo-SecureString "Password123!" -AsPlainText -Force
New-ADUser -Name "svc_web" -SamAccountName "svc_web" -AccountPassword $pass -Enabled $true

Set-ADUser -Identity "svc_web" -Add @{
    'msDS-AllowedToDelegateTo'=@(
        'CIFS/happy-mgmt.happy.corp',
        'CIFS/happy-mgmt'
    )
}

# Protocol Transition: permite S4U2Self sin necesitar la contraseña del usuario a impersonar
Set-ADAccountControl -Identity "svc_web" -TrustedToAuthForDelegation $true

Set-ADUser -Identity "svc_web" -ServicePrincipalNames @{
    Add='HTTP/happy-websvc.happy.corp'
}
```

**Resource-Based Constrained Delegation (RBCD)** — a diferencia de las anteriores, aquí es el objeto destino el que decide quién puede delegar hacia él. Si un atacante tiene `GenericWrite`/`GenericAll` sobre un objeto equipo, puede configurar esa relación de confianza él mismo.
```powershell
dsacls "CN=HAPPY-DC,OU=Domain Controllers,DC=happy,DC=corp" /G "$((Get-ADUser n3oari).SID.Value):GW"
```

### SQL Server mal integrado en el dominio

Los servidores SQL unidos al dominio son un objetivo recurrente: suelen correr con cuentas de servicio con privilegios de más, aceptar autenticación mixta con contraseñas débiles, y en entornos con varios dominios/forests suelen tener enlaces entre sí (linked servers) que abren rutas de movimiento lateral.

Instalamos SQL Server en `happy-sql`, usando la cuenta `svc_sqlsvc` ya creada (la misma con SPN registrado para Kerberoasting):

```powershell
setup.exe /Q /ACTION=Install /FEATURES=SQLEngine /INSTANCENAME=MSSQLSERVER /SQLSVCACCOUNT="HAPPY\svc_sqlsvc" /SQLSVCPASSWORD="Password123!" /SQLSYSADMINACCOUNTS="HAPPY\Administrator" /SECURITYMODE=SQL /SAPWD="Password123!" /IELCID=1033 /IACCEPTSQLSERVERLICENSETERMS
```

```powershell
netsh advfirewall firewall add rule name="SQL Server" protocol=TCP dir=in localport=1433 action=allow
# Habilitar TCP/IP en SQL Server Network Configuration → Protocols for MSSQLSERVER → TCP/IP → IPAll → puerto 1433
# Reiniciar el servicio tras el cambio:
net stop MSSQLSERVER; net start MSSQLSERVER
```

**Cuenta Domain Admin kerberoasteable** — además del servicio SQL normal, registramos un SPN sobre `svc_dcadmin` (la cuenta que ya es miembro de Domain Admins). Es uno de los hallazgos más graves posibles: un Kerberoasting exitoso contra esta cuenta compromete el dominio entero.
```powershell
Set-ADUser -Identity svc_dcadmin -ServicePrincipalNames @{Add='MSSQLSvc/happy-mgmt.happy.corp:1433'}
```

**Impersonación dentro de SQL Server** — concedemos a `n3oari` un login de Windows con permiso de `IMPERSONATE` sobre `sa`, una misconfiguration típica que permite escalar de un login con pocos privilegios al rol de administrador del motor SQL.
```sql
USE master

CREATE LOGIN [HAPPY\n3oari] FROM WINDOWS;
GRANT IMPERSONATE ON LOGIN::sa TO [HAPPY\n3oari];
```

**Linked server hacia el dominio hijo** — un servidor enlazado con credenciales embebidas hacia `us-happy-sql`, en `us.happy.corp`. Es la vía clásica de movimiento lateral entre dominios de un mismo forest a través de SQL Server.
```sql
EXEC sp_addlinkedserver 
    @server = 'us-happy-sql',
    @srvproduct = '',
    @provider = 'MSOLEDBSQL',
    @datasrc = 'us-happy-sql.us.happy.corp',
    @provstr = 'TrustServerCertificate=yes'

EXEC sp_addlinkedsrvlogin 
    @rmtsrvname = 'us-happy-sql',
    @useself = 'false',
    @rmtuser = 'sa',
    @rmtpassword = 'Password123!'
```

### Plantillas de certificado (ADCS) mal configuradas

Active Directory Certificate Services es una de las superficies de ataque menos vigiladas en entornos empresariales reales, porque sus misconfiguraciones viven en plantillas de certificado que rara vez se auditan tras su creación inicial.

Instalamos el rol en `happy-dc`:

```powershell
Install-WindowsFeature AD-Certificate -IncludeManagementTools -IncludeAllSubFeature
Install-AdcsCertificationAuthority -CAType EnterpriseRootCA -CACommonName "happy-corp-CA" -KeyLength 2048 -HashAlgorithmName SHA256 -CryptoProviderName "RSA#Microsoft Software Key Storage Provider" -Force
Restart-Computer -Force
```

**Plantilla con sujeto proporcionado por el solicitante**

El fallo de diseño aquí es dejar que el propio solicitante especifique el nombre del sujeto del certificado — con eso, un usuario normal puede pedir un certificado indicando que representa a cualquier otra identidad, incluida una cuenta con privilegios de administrador. Quien conozca la clasificación de Certified Pre-Owned reconocerá esto como **ESC1**.

Creamos una plantilla `WebAuth-ClientCert` a partir de `certtmpl.msc` (duplicando la plantilla "User", versión Windows Server 2003 Enterprise):

- **General**: nombre `WebAuth-ClientCert`, validez 1 año, publicar en Active Directory.
- **Subject Name**: marcar **"Supply in the request"** — el punto clave de esta misconfiguration.
- **Security**: `Domain Users` con permisos de `Read` + `Enroll`.
- **Issuance Requirements**: sin aprobación del gestor de la CA, 0 firmas autorizadas.
- **Extensions → Application Policies**: incluir `Client Authentication`.

Publicamos la plantilla en la CA:
```markdown
certsrv.msc → Certificate Templates → clic derecho → New → Certificate Template to Issue → WebAuth-ClientCert
```

**Plantillas encadenadas mediante un rol de "agente"**

Una segunda variante, más sutil: en vez de dejar que el usuario indique directamente el sujeto, se permite que una cuenta con un rol de "agente de inscripción" solicite certificados **en nombre de otra identidad**. Si ese rol de agente es fácil de conseguir, la cadena completa equivale a poder pedir un certificado como cualquier usuario del dominio. En la misma clasificación, esto corresponde a **ESC3**.

Plantilla 1 — `HelpDesk-Agent` (concede el rol de agente):
- **Security**: `Domain Users` con `Read` + `Enroll`.
- **Extensions → Application Policies**: eliminar las políticas por defecto y añadir `Certificate Request Agent` (OID `1.3.6.1.4.1.311.20.2.1`).

Plantilla 2 — `HelpDesk-UserAuth` (la que se solicita "en nombre de otro"):
- **Subject Name**: "Build from Active Directory" (por defecto).
- **Security**: `Domain Users` con `Read` + `Enroll`.
- **Extensions → Application Policies**: `Client Authentication`.
- **Issuance Requirements**: 1 firma autorizada, con política de aplicación `Certificate Request Agent` — esto obliga a que quien la solicite ya tenga el rol de agente de la Plantilla 1.

Publicamos ambas:
```markdown
certsrv.msc → Certificate Templates → clic derecho → New → Certificate Template to Issue → HelpDesk-Agent
certsrv.msc → Certificate Templates → clic derecho → New → Certificate Template to Issue → HelpDesk-UserAuth
```

## Fase 5 — Validación del entorno

Antes de empezar a practicar sobre el laboratorio, conviene confirmar que cada misconfiguration quedó realmente aplicada. Un smoke test rápido, con herramientas nativas de Windows — sin recurrir a PowerView ni BloodHound, para mantener la línea de "esto es configuración, no explotación":

```powershell
# Shares expuestos
Get-SmbShare | Select Name, Path

# Unconstrained Delegation
Get-ADComputer -Filter * -Properties TrustedForDelegation | Select Name, TrustedForDelegation

# Constrained Delegation
Get-ADUser -Filter * -Properties msDS-AllowedToDelegateTo, TrustedToAuthForDelegation |
    Where-Object { $_.'msDS-AllowedToDelegateTo' } |
    Select Name, 'msDS-AllowedToDelegateTo'

# RBCD configurado
Get-ADComputer happy-dc -Properties msDS-AllowedToActOnBehalfOfOtherIdentity

# Cuentas con SPN (kerberoasteables)
Get-ADUser -Filter * -Properties ServicePrincipalName | Where-Object { $_.ServicePrincipalName } | Select Name, ServicePrincipalName

# Cuentas sin preautenticación (AS-REP roasteables)
Get-ADUser -Filter * -Properties DoesNotRequirePreAuth | Where-Object { $_.DoesNotRequirePreAuth } | Select Name

# GPOs enlazadas y su ámbito
Get-GPO -All | Select DisplayName, Id
Get-GPInheritance -Target "OU=Servers,DC=happy,DC=corp"

# Plantillas de certificado publicadas
certutil -CATemplates
```

Si cada uno de estos comandos devuelve lo esperado, el entorno está listo para empezar a practicar sobre él.

## Conclusión

Este laboratorio no pretende ser un CTF con una única solución. Es una base para construir tu propio path de ataque, entendiendo desde dentro por qué cada misconfiguration existe y cómo se llega a ella — no solo cómo explotarla.

El enfoque manual, frente a un entorno ya montado tipo GOAD, obliga a pasar por cada decisión que normalmente toma un administrador real: qué permisos concede, qué cuenta usa para qué servicio, qué GPO aplica dónde. Ese contexto es precisamente lo que se pierde cuando el entorno ya viene resuelto.

Queda abierto a seguir creciendo: un segundo forest con su propio trust, más combinaciones de misconfiguraciones, o simplemente tu propia versión con otros nombres y otro layout. Si construyes tu propia variante, siéntete libre de tomar esto como punto de partida.
