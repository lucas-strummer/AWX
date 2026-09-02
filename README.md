# Gestión de `svc_ansible` desde AWX

La solución separa la generación de credenciales del bootstrap de servidores.
Ninguna clave SSH se almacena en Git:

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
servidores nuevos: crea o actualiza el mismo usuario, instala siempre la misma
clave pública y valida `/etc/sudoers.d/svc_ansible` con `visudo`.

Cuando todos los hosts estén preparados, los Job Templates normales pueden usar
la credencial `svc_ansible SSH` con escalamiento de privilegios.

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
