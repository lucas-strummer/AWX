# Gestión de `svc_ansible` desde AWX

Este repositorio también contiene playbooks operativos reutilizables importados
en [`playbooks/`](playbooks/). La importación fue sanitizada antes de publicarse:

- no incluye inventarios reales, direcciones IP ni reportes generados;
- no almacena hashes de contraseñas ni claves SSH privadas;
- las credenciales se reciben mediante variables protegidas de AWX o Ansible
  Vault;
- `.gitignore` bloquea inventarios, claves privadas, vaults y resultados habituales.

Los valores mínimos para los playbooks de usuario y clave son
`managed_user`, `managed_user_groups`, `managed_user_password_hash` y
`managed_user_public_key`, según corresponda. Los valores sensibles deben
inyectarse desde credenciales protegidas y nunca desde archivos versionados.

La solución separa la generación de credenciales del bootstrap de servidores.
Ninguna clave SSH privada se almacena en Git:

- la clave privada queda cifrada en una credencial `Machine` de AWX;
- la clave pública queda cifrada en una credencial personalizada de AWX;
- el par temporal se elimina siempre del Execution Environment.

## 1. Provisionar las credenciales una sola vez

El Job Template de `provision_svc_ansible_credentials.yml` se ejecuta solamente
en `localhost`, sin `become` y sin inventario de servidores.

Debe tener adjunta una credencial personalizada que inyecte:

```text
CONTROLLER_HOST=https://awx.example.com
CONTROLLER_OAUTH_TOKEN=<token con permiso para administrar credenciales>
```

Variables del Job Template:

```yaml
svc_ansible_confirm_generation: true
svc_ansible_organization: Default
```

La confirmación es obligatoria porque cada ejecución genera una clave nueva. El
playbook crea o actualiza:

- `svc_ansible SSH`: credencial `Machine` con usuario y clave privada;
- `svc_ansible Public Key Type`: tipo de credencial personalizado;
- `svc_ansible Public Key`: credencial que inyecta `svc_ansible_public_key`.

Después del alta inicial, desactive o elimine el Job Template de provisión para
evitar rotaciones accidentales. Para rotar una clave, vuelva a habilitarlo y
ejecútelo intencionalmente; después ejecute inmediatamente el bootstrap sobre
todos los hosts.

## 2. Ejecutar el bootstrap idempotente

El Job Template de `bootstrap_svc_ansible.yml` debe usar:

- el inventario que contiene los servidores Linux de destino;
- una credencial `Machine` administrativa existente durante el alta inicial;
- la credencial `svc_ansible Public Key`;
- `Privilege Escalation`, si la credencial administrativa lo requiere.

El playbook usa `hosts: all`: AWX determina los equipos mediante el inventario
seleccionado. Para ejecutar sobre un subconjunto, complete el campo `Límite` del
Job Template o actívelo como opción de pregunta al ejecutar.

El bootstrap no genera claves. Puede ejecutarse repetidamente y también sobre
servidores nuevos. Primero consulta la base de usuarios del sistema: si
`svc_ansible` no existe, lo crea; si ya existe, conserva sus atributos actuales.
En ambos casos instala la clave pública y valida
`/etc/sudoers.d/svc_ansible` con `visudo`.

Cuando todos los hosts estén preparados, los Job Templates normales pueden usar
la credencial `svc_ansible SSH` con escalamiento de privilegios.

## Cuenta `operador` para JumpServer

El playbook
[`playbooks/jumpserver/provisionar_operador.yml`](playbooks/jumpserver/provisionar_operador.yml)
crea la cuenta Linux `operador`, bloquea su contraseña e instala la clave pública
versionada en
[`playbooks/jumpserver/files/jumpserver_operador_ed25519.pub`](playbooks/jumpserver/files/jumpserver_operador_ed25519.pub).

La clave autorizada sólo acepta conexiones cuyo origen sea JumpServer
(`10.100.100.84`) y deshabilita agent forwarding, port forwarding, X11 forwarding
y los comandos de inicio definidos en `~/.ssh/rc`. Se conserva la asignación de
PTY necesaria para la terminal interactiva.

Configure un Job Template con:

- el inventario de servidores Linux de destino;
- la credencial `Machine` de `svc_ansible`;
- escalamiento de privilegios;
- el playbook `playbooks/jumpserver/provisionar_operador.yml`.

Ejecute primero sobre un único servidor mediante `Límite`. Confirme en sus logs
SSH que la conexión de JumpServer se vea con origen `10.100.100.84`; si existe
NAT, sobrescriba `jumpserver_source_ip` desde Extra Variables con la dirección
real observada.

`operador_authorized_key_exclusive` vale `true`, por lo que elimina del usuario
`operador` cualquier clave que no esté declarada por este playbook. Cámbielo a
`false` durante la primera ejecución si la cuenta ya existe y necesita auditar
sus claves antes de reemplazarlas.

La clave privada correspondiente debe cargarse como credencial de la cuenta
`operador` en JumpServer y mantenerse fuera de Git y AWX. El playbook no concede
permisos sudo a `operador`.

## Aprovisionamiento genérico de usuarios para JumpServer

El playbook
[`playbooks/jumpserver/provisionar_usuario.yml`](playbooks/jumpserver/provisionar_usuario.yml)
permite crear diferentes usuarios desde un único Job Template. El nombre, la
clave pública y la concesión de sudo se reciben como variables al iniciar el
trabajo.

Configure un Survey de AWX con estos campos:

| Pregunta | Tipo | Variable | Valor predeterminado |
| --- | --- | --- | --- |
| Nombre de usuario Linux | Text | `managed_user` | Sin valor |
| Clave pública SSH | Textarea | `managed_user_public_key` | Sin valor |
| ¿Conceder sudo completo? | Multiple Choice | `managed_user_sudo` | `false` |

Para la última pregunta, agregue las opciones `false` y `true`. Cuando se elige
`true`, el playbook crea `/etc/sudoers.d/jumpserver-<usuario>` con
`NOPASSWD: ALL`. Cuando se elige `false`, elimina solamente ese archivo; no
modifica otras reglas sudo ni los grupos de un usuario preexistente.

Las cuentas nuevas siempre se crean con la contraseña bloqueada. En cuentas
preexistentes, `managed_user_lock_password: false` permite conservar el estado
actual de su contraseña. La clave instalada es exclusiva y sólo se acepta desde
`10.100.100.84`; estos comportamientos pueden sobrescribirse mediante
`managed_user_authorized_key_exclusive`,
`managed_user_restrict_to_jumpserver` y `jumpserver_source_ip`.

Si el usuario ya existe, se conservan su grupo primario, comentario y directorio
personal, pero su shell se establece explícitamente en `/bin/bash`. El playbook
también administra su bloqueo de contraseña, su clave autorizada y únicamente el
archivo sudoers con prefijo `jumpserver-`.

## Execution Environment

Para `ansible-core 2.15`, las colecciones están fijadas a versiones compatibles
en `collections/requirements.yml`:

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

## Consideraciones de seguridad

- `svc_ansible_authorized_key_exclusive: true` elimina otras claves autorizadas
  del usuario `svc_ansible`. Use `false` si necesita conservarlas.
- El token de API debe pertenecer a una cuenta de servicio con los permisos
  mínimos necesarios y debe inyectarse como secreto, nunca como extra variable.
- La clave pública se marca como secreta para que AWX no la muestre en jobs,
  aunque criptográficamente no sea información privada.
- Una rotación no debe considerarse completa hasta actualizar correctamente todos
  los hosts; mantenga una credencial administrativa de recuperación.
