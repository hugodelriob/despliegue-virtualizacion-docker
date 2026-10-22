# Caso Práctico Tipo Empresa: Auditoría y Despliegue Seguro

## 1. Descripción del Proyecto y Arquitectura

Este proyecto implementa una infraestructura perimetral desacoplada y segura mediante contenedores Docker, resolviendo el problema operativo de publicar múltiples servicios web sobre el puerto público 80 mediante un **Proxy Inverso (Nginx)** [1, 3].

### Componentes de la Red:
1. **Edge Router (`srv-nginx-proxy`):** Servidor Nginx expuesto al exterior en el puerto `80:80` del host anfitrión. Encargado del enrutamiento por nombre de dominio, filtrado de seguridad y terminación de tráfico perimetral [1, 3].
2. **Backend Aislado (`srv-apache-app`):** Servidor Apache que aloja la aplicación interna. **No expone ningún puerto al host** y solo es accesible dentro de la red privada virtual `red-corp` a través de Nginx [2, 3].
3. **Red Privada (`red-corp`):** Red interna de tipo `bridge` que aísla la comunicación backend del exterior [2].

### Diagrama de la Arquitectura
```text
  [ Cliente Exterior / Browser ]
                │
                │ (HTTP / Puerto 80)
                ▼
┌────────────────────────────────────────────────────────┐
│ HOST ANFITRIÓN (0.0.0.0:80)                            │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │ Contenedor: srv-nginx-proxy (Nginx 1.25)         │  │
│  │ - Hardening OWASP (server_tokens off)            │  │
│  │ - Volúmenes en Solo Lectura (:ro)                │  │
│  └───────────────────┬──────────────────────────────┘  │
│                      │                                 │
│                      │ Red Privada Docker (red-corp)   │
│                      ▼                                 │
│  ┌──────────────────────────────────────────────────┐  │
│  │ Contenedor: srv-apache-app (Apache 2.4)          │  │
│  │ - Endpoint /privado/ protegido con X-Auth-Token  │  │
│  │ - Trazabilidad de IP Real (X-Forwarded-For)      │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
```

## 2. Instrucciones de Despliegue y Ciclo de Vida
Requisitos Previos
Añadir la resolución de nombres local en el archivo de hosts (/etc/hosts en Linux/macOS o C:\Windows\System32\drivers\etc\hosts en Windows):
```text
127.0.0.1 portal.empresa.local app-gestion.empresa.local
```
### Comandos de Despliegue
```text
# 1. Iniciar la infraestructura en segundo plano
docker compose up -d

# 2. Verificar el estado de los contenedores y aislamiento de puertos
docker compose ps

# 3. Consultar las trazas de log en tiempo real
docker compose logs -f

# 4. Detener la infraestructura
docker compose down
```

## 3. Evidencias de Verificación y Auditoría
## A. Prueba de Resolución de Dominios (curl -I)
Virtual Host Estático (portal.empresa.local):
```text
curl -I http://portal.empresa.local
```

<img width="631" height="257" alt="image" src="https://github.com/user-attachments/assets/88c4238a-b243-4306-a937-7cde3ef60fd3" />

Virtual Host Proxy (app-gestion.empresa.local):
```text
curl -I http://app-gestion.empresa.local
```

<img width="583" height="192" alt="image" src="https://github.com/user-attachments/assets/c69dabdb-4907-4a57-bb95-739b45e59784" />

## B. Auditoría de Seguridad en Endpoint Restringido (/privado/)
Acceso Denegado (Sin cabecera de autenticación):
```text
    curl -i -H "Host: app-gestion.empresa.local" http://localhost/privado/
```

<img width="686" height="277" alt="image" src="https://github.com/user-attachments/assets/0dd7d416-1e82-4545-ad1c-bbb61e11cb66" />

Resultado: HTTP/1.1 403 Forbidden (Acceso rechazado por Apache).

Acceso Permitido (Con cabecera X-Auth-Token: DAW2026):
```text
curl -i -H "Host: app-gestion.empresa.local" -H "X-Auth-Token: DAW2026" http://localhost/privado/
```

<img width="736" height="225" alt="image" src="https://github.com/user-attachments/assets/2db9b12f-a4d2-48cc-9736-6633853c2d12" />

Resultado: HTTP/1.1 200 OK (Acceso concedido).

## C. Auditoría de Trazabilidad en Logs (apache/logs/access.log)
Comprobación en los ficheros de log de la IP real del cliente registrada (127.0.0.1) junto con los códigos HTTP 403 y 200:

<img width="850" height="209" alt="image" src="https://github.com/user-attachments/assets/bf25c23a-310f-4619-bbdb-bed7e13aeb30" />
