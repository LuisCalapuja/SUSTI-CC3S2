### 1. Justifica por qué IaC + supply chain security es indispensable en este sistema. 

La IaC no es suficiente para un entorno clinico, ya que nos da:
- Estado deseado (lo que se despliega)
- Revision y Control de Cambios (PRs)
- Reproductibilidad

Esto nos indica que si por alguna razon la IaC fallara en un punto, no podriamos probar lo que realmente corrio (codigo), sino lo que se declaro como tal. Ya que es unicamente DECLARATIVO.

Con Supply Chain Security podriamos generar **firmas** criptograficas de artefactos, que nos ayudan como PRUEBA de:
- Integridad
- Autenticidad
- Trazabilidad
- **No repudio**

La diferencia con el tag consiste en que este es solo un nombre que se puede modificar, en cambio el la prueba que realizamos al codificar usando sha256 (digest) nos garantiza integridad del contenido exacto. Asi podemos afirmar que la imagen ejecutada corresponde al digest firmado por la clinica.


El **SBOM** es como un informe de componentes (liberias, versiones, hashes). Nos ayuda a identificar si alguna dependencia tenia alguna vulneravilidad el dia que se realizo el diagnostico. Asi podemos realizar las siguientes operaciones:
- Bloquear despliegues si SBOM reporta una CVE critica.
- Demostrar que se uso una libreria comprometida.

La **provenance** es un documentos firmado criptograficamente,describe el proceso de construccion de una artefacto. Funciona como una cadena de custodia criptografica del software, demuestra como se produjo un artefacto y bajo que comandos. 

El provenance incluye:
- URL del repositorio
- Commit hash exacto
- El sistema de CI que ejecuto el build (Github Actions/ Workflow especifico/ Job o run ID)
- Entorno de build: Sistema Operativo, Toolchain (Docker version, Python, CUDA, etc)

#### Disputa tecnica realista.
Un paciente es diagnosticado erroneamente, y el paciente entabla un juicio contra la clinica por este motivo. Sin Supply Chain, no hay forma de probar el contenido desplegado (no hay digest), no hay firma verificable, no se generaron los SBOM firmados y mucho menos un provenance que garantice la integridad durante la contruccion de los artefactos. Seria un caso en el cual no podriamos demostrar con PRUEBAS que se realizo correcta o incorrectamente el diagnostico.

### 2. Diseño de la cadena de evidencia tecnica
CODIGO FUENTE
- Git con commits firmados (GPG)
- Tag inmutables (vX.Y.Z)
- Ramas protegidas

**Artefacto**: commit hash firmado

BUILD
- Pipeline reproducible (GitHub Actions / GitLab CI)
- Build hermetico (no latest)
- Generacion automatica de SBOM (syft) y provenance (SLSA)

**Artefactos**: `image@sha256...`, `sbom.json`, `provenance.intoto.jsonl`

FIRMA
- Firma de imagen con cosign
- Firma del SBOM
- Se verifica con:
    ```bash
    cosign verify --key cosign.pub image@sha256:...
    ```
IaC VERSIONADA
- Manifiestos Kubernetes en Git
- Hash del manifiesto referenciado en pipeline
- No se aplica nada manualmente.

DESPLIEGUE
- Registro del despliegue (audit-service)
- Metadata: imagen digest, commit, operador (OIDC), timestamp

### 3. Define un estándar extremo de IaC limpia para entorno clínico: 
REGLAS OBLIGATORIAS:
- Commits firmados (si no, se rechaza)
- PR con doble aprobacion (tecnica + clinica/seguridad)
- Gates de seguridad (SAST, IaC scanning, Policy-As-Code)
- Secretos rotables (Por entorno, rol y TTL corto)
- Variables tipadas (Validación fuerte como Terraform validation {})

ANTIPATRONES SILENCIOSOS:
- Tags mutables en artefactos
```bash
image: inference-service:latest
image: inference-service:v2.1.0
```
¿Que pasa?: El tag puede reasignarse a otra imagen distinta sin dejar rastro
Riesgo: No se puede probar que modelo exacto corrio cuando se emitio un diagnostico.

- Artefactos compartidos entre experimentacion y produccion
Mismo bucket/registry para:
    - Modelos experimentales
    - Modelos clinicos certificados
¿Que pasa?: No hay frontera tecnica entre investigacion y produccion.
Riesgo: Un modelo experimental puede ser desplegado por accidente.

### 4. Selecciona y aplica patrones:
#### Builder para imagenes
- Dockerfiles multi-stages certificados:
    - Nos permite reducir errores manuales.
    - Facilidad en auditoria (etapas controladas).

#### Prototype para entornos de prueba
- Clon exacto del entorno de prueba:
    - Nos permite validar rollback sin tocar produccion.
    - Reduce tiempo de recuperacion.
#### Composite para observabilidad clinica
- Metricas, logs, trazas por paciente.
    - Nos permite vista unificada por auditoria
    - Reduce costo cognitivo y de investigacion post-incidente

### 5. Rediseña las dependencias entre servicios usando:
Creamos la siguiente arquitectura:
#### Servicios
- acquisition-service: captura señales biomedicas y empaqueta observaciones.
- inference-service: ejecuta inferencia ML local sobre observaciones validas.
- audit-service: actura como Mediator (coordinador clinico y cadena de evidencias)

#### Flujo
1. acquisition-service publica `ObservationRecorded(vX)`.
2. audit-service valida y registra evidencia.
3. audit-service emite comando `RunInference(vX)` hacia inference-service.
4. inference-service devuelve `InferenceResult(vX)`
5. audit-service finaliza: escribe "Diagnostico emitido" con hashes, versiones y timestamps.

#### DIP aplicado:
Los servicios no dependen de implementaciones concretas ni endpoints específicos, sino de abstracciones (contratos) estables. Por ejemplo:
- acquisition-service depende de una abstracción: ClinicalEventBus o AuditSink
- inference-service depende de: InferenceCommandHandler/ObservationRepository (abstracto)
- audit-service depende de interfaces: ObservationStore, InferenceGateway, EvidenceLedger

#### Mediator (audit-service como coordinador clinico)
audit-service orquesta y garantiza:
- orden de eventos
- validacion de contratos versionados
- persistencia de evidencia (append-only)
- politica clinica de degradación ("safe mode")

#### Contratos versionados
Versionar un contrato significa que:
- Se define el formato oficial de los datos (del diagnostico)en la actual version, eso puede cambiar mañana.
- Da la posibilidad de demostrar que version se uso en un diagnostico.

#### Invariantes de datos
- Inmutabilidad: Ningun diagnostico puede modificarse, cualquier modificacion genera un nuevo registro.
- Trazabilidad: Los datos deben estar asociados a: id unico, timestamp, version de contrato, version de imagen.
- Integridad verificable: Observaciones y resultados incluyen hash.
- No diagnostico sin evidencia

#### Limites de responsabilidad
acquisition-service:
- captura y valida señales biomedicas.
- publica observaciones segun contrato versionado
- No ejecuta inferencia ni gestiona evidencia

inference-service
- ejecuta inferencia sobre solicitudes validas
- devuelve resultados con metadata tecnica
- no decide flujo clinico ni persiste evidencia

audit-service (Mediator)
- coordina el flujo clinico completo
- valida contratos y versiones
- registra evidencia legal y estado del diagnostico

#### Que ocurre ante degradacion parcial
Falla de inference-service
- Se detienen diagnosticos automaticos
- Las observaciones se almacenan y registran
- Se activa alerta clinica para revision manual

Falla de audit-service
- El sistema entra en fail-closed para diagnosticos
- Se permite solo captura temporal cifrada
- Se genera alerta por riesgo legal

Falla de acquisition-service
- No hay nuevos datos
- No se ejecuta inferencia
- El sistema permanece en estado seguro

Operación sin conectividad
- Todo el flujo funciona localmente
- La sincronizacion externa se difiere sin afectar diagnosticos locales

Las invariantes garantizan integridad, la separacion de responsabilidades evita acoplamiento, y la degradacion parcial prioriza seguridad sobre disponibilidad.

### 6. Diseña un plan de pruebas de IaC y plataforma que incluya: 
#### Seguridad IaC
- Validacion y lint de manifests
- Escaneo de IaC para detectar secretos, privilegios excesivos y configuraciones inseguras
- Policy-as-Code (OPA) en CI y admision
- Bloqueo de imagenes sin digest, sin firma o con latest

#### Pruebas con rollback con datos sensibles
- Rollback de servicios sin reprocesar ni exponer datos
- Verificar que datos permanecen cifrados tras revertir
- Registrar rollback en auditoria
- Validar que secretos y permisos no reaparecen

#### e2e con teardown seguro
- Flujo completo: adquisicion > auditoria > inferencia > registro
- Teardown obligatorio:
    - borrar volumenes, secretos y jobs temporales
    - verificar ausencia de PII o datos residuales

#### Políticas OPA específicas del dominio salud
- Prohibir latest y exigir digest
- Denegar contenedores privilegiados y host access
- NetworkPolicy default-deny obligatoria
- Cifrado obligatorio para datos clinicos
- RBAC minimo y etiquetas de trazabilidad obligatorias

#### Lista 5 defectos críticos detectables
- Uso de latest o imagen sin digest.
- Contenedores privilegiados/host access
- Falta de NetworkPolicy default-deny
- Secretos expuestos en manifests
- RBAC excesivo

#### Lista 2 no detectables automáticamente
- Error o sesgo clinico del modelo
- Abuso de sistema por usuario autorizado

### 7. Despliegue final
#### Docker con aislamiento fuerte
- Contenedores no root, filesystem read-only
- Drop de capacidades Linux, seccomp/AppArmor
- Sin hostNetwork, hostPID ni hostPath
- Imagenes minimas y pinning por digest (no latest)

#### Kubernetes con zero-trust real
- mTLS entre servicios (identidad por workload)
- ServiceAccount por microservicio con RBAC minimo
- Pod Security restricted
- Admission control con OPA (bloquea configs inseguras)
- Secretos montados, rotables, nunca hardcodeados

#### NetworkPolicies por identidad
- Default-deny en namespace clinico.
- Permisos explícitos por labels/ServiceAccount:
    - acquisition > audit
    - audit > inference
    - inference < solo audit
- Egress bloqueado o allowlist minimo.

#### Estrategia de actualización conservadora (justificada)
- Blue/Green con ventanas de cambio y doble aprobacion
- No canary: evita que pacientes distintos sean diagnosticados por versiones distintas y simplifica responsabilidad legal

#### Alertas clínicas (no técnicas)
- Diagnostico no disponible > revisión manual
- Resultados fuera de rango clinico esperado
- Diagnostico sin evidencia completa
- Modelo no certificado en ejecucion
- Cola excesiva de casos pendientes

#### Evidencia mínima de no-repudio
- Digest de imagen por diagnostico
- Firma verificada
- SBOM asociado
- Provenance del build
- Registro append-only del audit-service con version y timestamps