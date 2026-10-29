# Passbolt self-hosted: gestor de contraseñas con recuperación probada

Passbolt Community Edition montado en casa para un equipo pequeño. Certificado real, red privada en vez de internet, respaldo externo cifrado y verificado, y un procedimiento de recuperación que se probó en un apagón de verdad, no en un diagrama.

![Arquitectura](media/arquitectura.svg)

## Por qué esto no es como montar cualquier servicio

Si un gestor de contraseñas compartido se cae, el equipo entero se queda sin acceso a todo: correos, bancos, paneles, llaves SSH. Y si lo levantas de cero con prisa, puede arrancar con la llave equivocada y dejar a cada persona fuera de sus propios secretos. Es el único servicio de la casa donde una restauración mal hecha no da un error, da un silencio.

Los servicios en la nube resuelven esto con dinero y confianza. Montarlo uno mismo obliga a resolver a mano tres cosas que ellos esconden: la llave del servidor, el respaldo que de verdad restaura, y qué se hace cuando alguien pierde su recovery kit.

## Cómo quedó montado

```
extensión (cripto en el navegador del usuario)
        │  TLS — solo dentro del tailnet, sin exposición a internet
        ▼
   Caddy 2  (certificado real, no autofirmado)
        ▼
   Passbolt CE  ──►  MariaDB 10.11
        │                  │
   vol. gpg/jwt        vol. datos
        └────────┬─────────┘
                 ▼
        respaldo diario cifrado ──► 3 copias, 1 fuera del sitio
```

Cuatro contenedores, una red interna y los puertos publicados solo en la interfaz privada. La criptografía ocurre en el cliente: el servidor guarda sobres cifrados y nunca ve una contraseña en claro.

La decisión de fondo fue no exponerlo a internet. El gestor de secretos no necesita una IP pública, necesita estar en la misma red privada que el equipo, así que no hay superficie de ataque hacia fuera ni certificados que renovar a mano. El certificado sí es real, emitido por la autoridad del propio tailnet: uno autofirmado obliga a cada usuario a aceptar excepciones y, con el tiempo, a ignorar las advertencias del navegador. Y sobre todo lo demás: la llave del servidor es el sistema, todo lo demás es reemplazable.

## El apagón que lo validó

El servidor de producción se cayó con un corte de luz y no volvió a arrancar. La última copia externa era de las 04:15 de esa madrugada.

Levanté una instancia temporal en otro nodo desde esa copia cifrada: descargarla, descifrarla, importarla. Esto fue lo que quedó verificado:

| | |
|---|---|
| Usuarios | **11** |
| Recursos | **322** (259 activos) |
| Secretos | **330** |
| Llave del servidor | **idéntica a producción**, sin alarma de "llave cambiada" en la extensión |
| `healthcheck` | `OK` |

Ese último renglón es la diferencia entre recuperar el servicio y perder a los usuarios. Si la llave cambia, cada persona tiene que volver a enrolar su cuenta a mano.

## Las cuatro trampas que encontré

La primera es la peor. **El primer arranque puede generar una llave nueva en silencio:** el paquete trae la llave buena, pero si el directorio montado llega vacío el entrypoint genera una nueva sin avisar. El servidor queda funcionando y anunciando el fingerprint equivocado. Se detecta comparando `GET /auth/verify.json?api-version=v2` contra el fingerprint conocido, siempre, antes de decir que está arriba.

La segunda, los bind-mounts tienen que ser del uid 33 (`www-data`). Si no, `gpg` no puede crear su directorio y el contenedor entra en bucle de reinicio sin decir por qué.

La tercera, el certificado del tailnet solo se emite para el nombre del propio nodo. Desde el nodo temporal no pude pedir el certificado del nombre de producción, solo error 500 con "invalid domain". La salida fue publicar en el nombre del nodo que sí puede emitirlo, en un puerto libre.

La cuarta, había un volumen del código de la aplicación que no estaba en el respaldo. Lo correcto fue quitar ese montaje y usar el código que ya trae la imagen, que aplica las migraciones solo al arrancar.

## Un respaldo que se puede creer

Diario, cifrado y verificado, que no es lo mismo que configurado. Se empaqueta y cifra por conjunto, con rotación por antigüedad. Hay tres copias de cada elemento y una fuera del sitio. El verificador revisa que cada copia exista y cierra con `RESULTADO: OK`; si falta una, lo dice con nombre y hora. Y la restauración se probó de punta a punta: es exactamente lo que trajo de vuelta los 330 secretos.

## Recuperación de usuarios

Passbolt tiene un agujero conocido: si alguien pierde su recovery kit o su passphrase, sus secretos se van con él, incluso para el administrador del servidor.

En este despliegue existe un procedimiento para eso. Los kits quedan en custodia cifrada y la recuperación exige la participación de varios responsables: ninguna persona sola, ninguna llave sola. Cambiar la passphrase de un kit conserva el fingerprint original, así que el usuario no tiene que volver a enrolarse.

Ese procedimiento y la herramienta que lo acompaña no están en este repositorio. Es la parte que más trabajo tiene detrás y queda fuera de la exhibición.

## Stack

Passbolt CE, MariaDB 10.11, Caddy 2, Docker Compose, Tailscale para la red y los certificados, rclone para el respaldo cifrado externo y OpenPGP.js.

## Sobre el código

Este repositorio es documentación de exhibición: arquitectura, decisiones, el incidente real y sus resultados. No incluye código de despliegue, configuración funcional, dominios ni llaves.

© 2026 Javier Cuevas. Todos los derechos reservados. Disponible bajo licencia por contacto.
