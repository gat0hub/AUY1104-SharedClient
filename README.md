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
