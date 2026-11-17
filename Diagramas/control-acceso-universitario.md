# Sistema de control de acceso universitario

Diagramas de secuencia del flujo principal de acceso y del registro de visitantes.

## 1. Acceso de estudiantes, docentes y personal

```mermaid
sequenceDiagram
    autonumber
    actor U as Usuario
    participant CR as Credencial RFID/QR
    participant L as Lector y torniquete
    participant CT as Controlador de acceso
    participant SA as Servicio de Control de Acceso
    participant ID as Directorio institucional
    participant DB as BD de permisos y bitácora
    participant SG as Consola de seguridad

    U->>CR: Presenta la credencial
    CR->>L: Envía identificador seguro
    L->>CT: Credencial + ID de lector + sentido
    CT->>SA: POST /access/validate
    activate SA
    SA->>SA: Autentica el controlador con mTLS/JWT
    SA->>ID: Consulta titular, rol y vigencia
    ID-->>SA: Identidad y estado
    SA->>DB: Consulta permisos por zona y horario
    DB-->>SA: Reglas aplicables
    SA->>DB: Verifica anti-passback y aforo
    DB-->>SA: Estado de tránsito

    alt Acceso autorizado
        SA->>DB: Registra ACCESO_CONCEDIDO
        SA-->>CT: 200 OK, decisión PERMITIDO
        CT->>L: Activa relé de apertura
        L-->>U: Luz verde y paso libre
        U->>L: Cruza el punto de acceso
        L->>CT: Señal de paso confirmado
        CT->>SA: Evento de tránsito completado
        SA->>DB: Actualiza aforo y auditoría
        opt Evento relevante
            SA->>SG: Notificación en tiempo real
        end
    else Acceso denegado
        SA->>DB: Registra ACCESO_DENEGADO y motivo
        SA-->>CT: 403 Forbidden, decisión DENEGADO
        CT->>L: Mantiene el bloqueo
        L-->>U: Luz roja y aviso
        opt Reintentos o credencial bloqueada
            SA->>SG: Alerta de seguridad
            SG-->>U: Solicita identificación en recepción
        end
    end

    deactivate SA
    Note over SA,DB: Cada evento queda en la bitácora para auditoría y reportes
```

### Participantes

| Participante | Responsabilidad |
|---|---|
| Usuario | Estudiante, docente o personal que solicita el acceso. |
| Credencial | Tarjeta RFID, código QR o dato biométrico. |
| Lector y torniquete | Lee la credencial, muestra el resultado y controla el paso físico. |
| Controlador de acceso | Dispositivo local que envía la solicitud y acciona el relé. |
| Servicio de Control de Acceso | Valida la solicitud, aplica reglas y toma la decisión. |
| Directorio institucional | Identidad, rol, vigencia y estado del usuario. |
| BD de permisos y bitácora | Zonas autorizadas, horarios, anti-passback, aforo y auditoría. |
| Consola de seguridad | Recibe alertas y atiende incidencias. |

### Reglas que se validan

- Credencial vigente y no bloqueada.
- Usuario activo y con rol habilitado.
- Permiso para la zona o edificio solicitado.
- Horario permitido según el rol.
- Anti-passback: no reutilizar la misma entrada sin haber salido.
- Aforo máximo de la zona.
- Autenticidad del controlador mediante mTLS o JWT.

## 2. Acceso de visitantes

```mermaid
sequenceDiagram
    autonumber
    actor V as Visitante
    actor R as Recepcionista
    participant P as Portal de visitas
    participant H as Anfitrión
    participant SA as Servicio de Control de Acceso
    participant L as Lector QR y torniquete
    participant SG as Consola de seguridad

    V->>R: Se presenta en recepción
    R->>P: Registra datos de la visita
    P->>H: Solicita aprobación
    alt El anfitrión aprueba
        H-->>P: Aprueba la visita
        P->>SA: Emite pase QR temporal
        SA-->>P: Pase con vigencia y zona
        P-->>R: Genera credencial QR
        R-->>V: Entrega el pase
        V->>L: Escanea el código QR
        L->>SA: Valida el pase
        SA-->>L: Decisión PERMITIDO
        L->>L: Abre el torniquete
        SA->>SA: Registra entrada y hora
        opt Visita sensible o fuera de horario
            SA->>SG: Notifica para seguimiento
        end
    else El anfitrión rechaza o no responde
        P-->>R: Visita no autorizada
        R-->>V: Informa el rechazo
    end
```

## Notas de diseño

- **Tolerancia offline:** el controlador puede validar credenciales cacheadas y registrar eventos localmente si se pierde la conexión; luego los sincroniza.
- **Idempotencia:** cada evento lleva un identificador único para evitar duplicados al reintentar.
- **Seguridad:** comunicación cifrada, credenciales no almacenadas en claro y registro inmutable de eventos.
- **Privacidad:** solo se guardan los datos necesarios y se aplican políticas de retención.
- **Disponibilidad:** el servicio debe responder en menos de un segundo para no generar filas en los accesos.
