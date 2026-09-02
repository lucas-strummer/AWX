# Bootstrap de `svc_ansible` desde AWX

Este proyecto genera un par de claves SSH, almacena **la clave privada únicamente
en AWX** como una credencial de tipo `Machine`, elimina los archivos temporales y
configura en los servidores Linux:

- el usuario `svc_ansible`, con contraseña bloqueada;
- acceso SSH mediante la clave pública generada;
- administración con `sudo` sin contraseña.

> La credencial creada no puede usarse para el mismo bootstrap: el Job Template
> inicial debe conectarse con una credencial administrativa ya existente. En los
> trabajos posteriores se utiliza la nueva credencial `svc_ansible SSH`.

## Preparación

1. Instale las colecciones en la imagen de Execution Environment o inclúyalas al
   construirla:

   ```bash
   ansible-galaxy collection install -r collections/requirements.yml
   ```

2. Cree un inventario con un grupo llamado `linux` (puede usar
   `inventory.example.yml` como referencia).
3. Cree en AWX una credencial con acceso a la API y permiso para administrar
   credenciales. Exponga el token al módulo `awx.awx` mediante la variable de
   entorno `CONTROLLER_OAUTH_TOKEN`; configure también `CONTROLLER_HOST` y, si
   corresponde, `CONTROLLER_VERIFY_SSL`.
4. Cree un Job Template apuntando a `bootstrap_svc_ansible.yml`, seleccione el
   inventario y adjunte:
   - una credencial Machine administrativa existente para entrar a los Linux;
   - la credencial que expone el token de la API de AWX.

Variables opcionales del Job Template:

```yaml
svc_ansible_credential_name: svc_ansible SSH
svc_ansible_organization: Default
svc_ansible_sudo_nopasswd: true
```

Al terminar correctamente, cambie los Job Templates normales para usar la nueva
credencial `svc_ansible SSH` y active `Privilege Escalation` cuando corresponda.

## Consideraciones de seguridad

- `exclusive: true` hace que la clave generada sea la única clave autorizada para
  `svc_ansible`. Cámbielo a `false` si necesita conservar otras claves.
- La ejecución vuelve a generar y rotar la clave. Esto actualiza la credencial de
  AWX y la clave pública en todos los hosts alcanzados por el job.
- Si algún host queda fuera o falla durante una rotación, conservará la clave
  anterior. Corrija el host y repita el bootstrap usando la credencial anterior o
  una credencial administrativa de recuperación.
- Para limitar privilegios, reemplace `ALL` en el archivo sudoers por una lista de
  comandos explícita. La automatización general de Ansible suele requerir acceso
  amplio.
