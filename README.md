# AUY1104-SharedClient...

# Pipeline CI/CD: Despliegue Avanzado Blue-Green con Rollback Automático en Kubernetes

Este repositorio contiene la configuración del pipeline de Integración Continua (CI) y Despliegue Continuo (CD) para la API de ejemplo, utilizando **GitHub Actions** y **Kubernetes (K3s)**.

El diseño del pipeline cumple con los requerimientos estipulados en el encargo, aplicando homologies tecnológicas validadas por la docencia (K3s sobre AWS EC2 en sustitución de Amazon EKS, y Docker Hub en sustitución de Amazon ECR).

---

## 1. Arquitectura del Sistema

La arquitectura implementada se basa en un modelo modular y desacoplado, donde un repositorio de flujos de trabajo centralizado (`AUY1104-SharedWorkflows`) expone plantillas reutilizables mediante `workflow_call`, las cuales son consumidas por este repositorio cliente.

*   **Orquestador de Contenedores:** Clúster Kubernetes de un solo nodo utilizando **K3s** instalado sobre una instancia **AWS EC2** (Ubuntu Server, Región `us-east-1`).
*   **Registro de Imágenes (Registry):** **Docker Hub** público para el almacenamiento y versionamiento de las imágenes de contenedor de la API.
*   **Automatización (CI/CD):** **GitHub Actions Runners** conectados de forma segura al clúster de AWS mediante SSH.
*   **Seguridad de Red:** Acceso externo expuesto a través de un servicio de Kubernetes tipo `NodePort` en el puerto **30090**, configurado explícitamente en el *Security Group* de AWS.

---

## 2. Estrategia de Despliegue: Blue-Green

Para mitigar los riesgos asociados a caídas del servicio durante actualizaciones, se abandonó el modelo de recreación básico y se implementó una estrategia **Blue-Green**. 

### Mecanismo de Control de Tráfico
El desvío del tráfico de producción no se realiza duplicando infraestructura de red costosa, sino **manipulando los selectores nativos del objeto `Service` de Kubernetes** (`demo-api`).

1. **Detección Dinámica:** El pipeline consulta al clúster por SSH el color activo actual mediante:
   `kubectl get service demo-api -o jsonpath='{.spec.selector.version}'`
2. **Aislamiento:** El nuevo código se despliega siempre en el "slot" o color inactivo de manera 100% aislada. Los usuarios siguen consumiendo la versión antigua sin interrupciones.
3. **Inyección de Variables:** Usando la herramienta `envsubst`, se reemplazan dinámicamente las variables de entorno en los manifiestos YAML (`k8s/deployment.yaml`), asignando los tags de imagen y las etiquetas del slot objetivo (`blue` o `green`).

Activo usando blue y Despliegue usando green:
<img width="752" height="247" alt="image" src="https://github.com/user-attachments/assets/81be902e-8ddf-4c9b-9123-ac20d2402eed" />

Activo usando green y Despliegue usando blue:
<img width="930" height="308" alt="image" src="https://github.com/user-attachments/assets/bde10902-de11-42d3-9385-97bf720a7f13" />

Pods blue y  green corriendo simultáneamente:
<img width="916" height="160" alt="image" src="https://github.com/user-attachments/assets/f620439e-9c59-446b-b220-2e11417962d0" />
<img width="887" height="119" alt="image" src="https://github.com/user-attachments/assets/1a4e9452-3b8a-4d69-a9b7-371da30de007" />

Verificacion de health via putty y web:
<img width="915" height="254" alt="image" src="https://github.com/user-attachments/assets/35142682-b4ae-49a3-a03d-7db7ace844b6" />

<img width="780" height="395" alt="image" src="https://github.com/user-attachments/assets/b6c3f988-df98-4dd6-9520-b2593b840a4b" />



---

## 3. Preparación para el Caos: Remediación y Rollback Automático

El pipeline está diseñado bajo el principio de "defensa en capas", garantizando la alta disponibilidad incluso si se intenta desplegar código defectuoso (errores lógicos, latencias altas, fallos de configuración).

### ¿Cómo se activa la Remediación Automática?
[Falla detectada por ReadinessProbe]
│
▼
[Paso 6: Rollout Status Timeout] (Exit code 1)
│
▼
[Paso 7: Cambio de Tráfico] ──> ¡BLOQUEADO / CANCELADO!
│
▼
[Paso 8: Rollback Automático] ──> (Ejecutado por if: failure())
│
├──> Asegura el Service apuntando al color antiguo estable.
└──> Elimina el Deployment corrupto para limpiar el clúster.
1. **Validación de Salud (Fase Crítica):** Antes de conmutar el tráfico de los usuarios al nuevo color, el pipeline ejecuta un comando de bloqueo y monitoreo:
   `sudo k3s kubectl rollout status deployment/demo-api-${TARGET_SLOT} --timeout=60s`
   Este comando consulta los `readinessProbes` configurados en Kubernetes. Si los nuevos contenedores fallan o no responden en la ruta `/health`, el comando expira y el paso se marca con código de error (`Exit code 1`).
2. **Bloqueo del Switch:** Al fallar la validación, el paso de "Cambiar el tráfico al nuevo color" se omite por completo. Ningún usuario externo llega a ver el error.
3. **Ejecución del Rollback:** Utilizando la condicional de GitHub Actions `if: failure()`, se activa la etapa de remediación automática, la cual:
   * Aplica un parche de seguridad al `Service` para ratificar que siga apuntando al slot antiguo y saludable.
   * Ejecuta un borrado destructivo del deployment corrupto (`kubectl delete deployment`) para purgar los contenedores insalubres del clúster.
   
   Rollback de Ejemplo:
<img width="747" height="651" alt="image" src="https://github.com/user-attachments/assets/01a0f38d-7c5e-41c4-80f6-a12174cf1d64" />


## 4. Análisis de Impacto: MTTR y Costos Operativos en AWS

Como parte del diseño de esta arquitectura, se evaluó el impacto de la estrategia Blue-Green y su mecanismo de remediación en dos métricas críticas de operación y negocio:

### Minimización del MTTR (Mean Time To Recovery)
El mecanismo propuesto reduce el **MTTR** prácticamente a **segundos** y requiere **cero intervención humana**. Tradicionalmente, recuperarse de un despliegue defectuoso implica esperar alertas de monitoreo, diagnosticar el fallo y gatillar manualmente un despliegue de reversión. En nuestra implementación:
* **Prevención:** El error se intercepta en la fase de validación de salud (`readinessProbe` + `timeout`) antes de que se libere tráfico real.
* **Autocorrección instantánea:** Al gatillarse el condicional `if: failure()`, el script inyecta un `patch` al `Service` de Kubernetes. El sistema vuelve a su estado estable anterior a la velocidad en que la API de Kubernetes procesa una actualización de red, evitando tiempos muertos y mitigando la caída al instante.

### Impacto en los Costos Operativos (AWS)
La adopción de la estrategia Blue-Green conlleva un *trade-off* (compromiso) inherente respecto a los costos de infraestructura en la nube:
* **Consumo adicional temporal:** Durante la ventana de ejecución del pipeline, el clúster (operando sobre instancias EC2 de AWS) debe alojar el doble de carga, ya que requiere mantener operativos los contenedores *Blue* y *Green* simultáneamente. Esto implica picos temporales en el consumo de CPU y memoria RAM de las instancias.
* **Optimización mediante limpieza automatizada:** Para evitar sobrecostos por recursos inactivos o defectuosos, la etapa de remediación incluye una tarea destructiva explícita (`kubectl delete deployment demo-api-${TARGET_SLOT} --ignore-not-found=true`). Si la nueva versión falla, el clúster destruye los contenedores insalubres de forma automática e inmediata, liberando el cómputo de EC2 y asegurando que la factura de AWS no se infle por recursos estancados o huérfanos.

## 5. Impacto Arquitectónico en TechMarket

La implementación de este pipeline centralizado transforma el ciclo de vida del software para **TechMarket**, logrando dos beneficios fundamentales para el negocio:

*   **Aceleración de los Tiempos de Despliegue:** Al utilizar plantillas reutilizables (`workflow_call`), el equipo de TechMarket ya no necesita escribir scripts manuales ni conectarse por SSH para cada actualización. El proceso que antes tomaba minutos u horas de coordinación manual, ahora se ejecuta de forma paralela y automatizada en GitHub Actions en menos de 2 minutos.
*   **Reducción de Errores Manuales:** Se elimina el factor de error humano (como escribir mal un tag o borrar un servicio accidentalmente). Las validaciones obligatorias, como el análisis del clúster (`envsubst`) y los bloqueos por *timeout* (`kubectl rollout status`), aseguran que ningún código defectuoso pase a producción por un descuido humano.

---

## 6. Comparativa Técnica de Estrategias de Despliegue en Kubernetes

Para sustentar la elección de Blue-Green, se analizó el comportamiento de las distintas estrategias nativas y avanzadas dentro del ecosistema de Kubernetes:

| Estrategia | Comportamiento en Kubernetes | Ventajas | Desventajas / Riesgos |
| :--- | :--- | :--- | :--- |
| **All-in-once (Recreate)** | Borra los Pods de la versión anterior (`A`) completamente antes de crear los Pods de la nueva versión (`B`). | No hay problemas de compatibilidad de versiones. Muy fácil de configurar. | **Downtime inaceptable.** El servicio se cae mientras se levantan los nuevos contenedores. |
| **Rolling Update** | Estrategia por defecto en Kubernetes. Reemplaza los Pods uno por uno de forma gradual (ej. `maxSurge` y `maxUnavailable`). | Mantiene disponibilidad continua sin requerir doble infraestructura. | Los usuarios pueden recibir respuestas mixtas (versión A y B operando al mismo tiempo). Rollback lento. |
| **Canary** | Usa el `Service` o un *Ingress* para enviar un porcentaje pequeño de tráfico (ej. 10%) a los Pods nuevos y el resto a los antiguos. | Riesgo mínimo. Permite probar en producción con usuarios reales. | Alta complejidad técnica de red. Requiere monitoreo exhaustivo para decidir cuándo avanzar al 100%. |
| **Blue-Green (Elegida)** | Mantiene dos *Deployments* aislados (`blue` y `green`). El `Service` de K8s actúa como un interruptor de red (`spec.selector`) que cambia el tráfico al 100% en un instante. | Aislamiento total de versiones, Zero-Downtime y Rollback inmediato (un solo comando). | Mayor consumo de recursos computacionales, ya que exige mantener ambos entornos activos temporalmente. |
