# Recuperación de desastre: del respaldo al servicio, con checklist

Documento de trabajo real, sanitizado: nombres de nodos, direcciones y llaves sustituidos. Es el procedimiento que se siguió cuando el servidor de producción no arrancó.

## Escenario

Apagón en la zona, el equipo de producción no volvió a encender. Copia externa más reciente: la de las 04:15 de ese mismo día. Objetivo: devolver el acceso al equipo desde otro nodo, **sin tocar producción** y sin que nadie tenga que re-enrolar su cuenta.

## Procedimiento

1. **Bajar la copia cifrada externa más reciente** (no la local: si el nodo cayó, la copia local puede tener la misma edad o estar incompleta).
2. **Descifrar y abrir el paquete.** Debe contener, además del volcado de la base: llave GPG del servidor, par JWT y el `salt`/`passphrase` del entorno.
3. **Montar el stack** con los mismos nombres de servicio y una red interna nueva. Nada reutilizado del nodo caído.
4. **Restaurar la llave del servidor antes del primer arranque.** Si el directorio de GPG está vacío, el entrypoint genera una llave nueva en silencio.
5. **Importar el volcado** y esperar las migraciones (se aplican solas al arrancar).
6. **Verificar, no suponer:**
   - `GET /auth/verify.json?api-version=v2` devuelve **el fingerprint esperado**;
   - `healthcheck` en `OK`;
   - conteo de usuarios, recursos y secretos coincide con el respaldo.
7. **Publicar con TLS real** en un nombre de nodo válido y puerto libre.
8. **Avisar al equipo del cambio de origen.** La extensión guarda cuentas por origen: en la URL temporal pedirá re-agregar la cuenta con el recovery kit. El fingerprint del servidor es el mismo, así que no aparece la alarma de "llave cambiada".

## Checklist de verificación final

| Comprobación | Criterio de aceptación |
|---|---|
| Fingerprint del servidor | idéntico al de producción |
| Healthcheck | `OK` |
| Usuarios / recursos / secretos | iguales al respaldo |
| Cuenta del administrador | activa y con su llave |
| Rollback probado | `docker compose down -v` + borrado del directorio |

## Trampas que costaron tiempo

- **Bind-mounts con dueño equivocado.** Deben pertenecer al uid 33 (`www-data`); si no, `gpg` no crea su directorio y el contenedor reinicia en bucle sin mensaje útil.
- **Certificado de tailnet.** Solo se emite para el nombre del propio nodo: desde el nodo temporal no se puede pedir el de producción (`500 invalid domain`).
- **Volumen de código no incluido en el respaldo.** Quitar el montaje y usar el código de la imagen evita restaurar una versión vieja de la aplicación.
- **Puertos ocupados.** Antes de elegir puerto, comprobar cuáles están tomados en el nodo destino; el 443 casi nunca está libre en una máquina que ya sirve algo.

## Rollback

Un comando por cosa, escrito antes de empezar:

```
docker compose down -v
rm -rf <directorio-del-montaje-temporal>
```

El certificado y los datos viven dentro del directorio, así que desaparecen con él. Producción no se toca en ningún paso de este procedimiento.
