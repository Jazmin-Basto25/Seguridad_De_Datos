###  **Instalación de Kubernetes Goat**

Ejecuta los siguientes comandos:

```bash
git clone https://github.com/madhuakula/kubernetes-goat.git
cd kubernetes-goat
bash setup-kubernetes-goat.sh
```

### 🔍 **Verificar que los Pods estén corriendo**

```bash
kubectl get pods
```

Si todos los Pods están en estado **Running**, puedes empezar a ejecutar los escenarios.


# ## **1. Sensitive Keys in Codebases (Escenario 1)**

###  **Descripción de la Vulnerabilidad**

La aplicación **build-code-deployment** estaba configurada de tal manera que el directorio **.git** del repositorio interno era accesible públicamente a través de una solicitud HTTP.
Esto permite que un atacante descargue el repositorio completo y su historial de commits, incluso si los archivos ya fueron eliminados del branch actual.


### 🛠️ **Detalle del Proceso de Explotación**

#### **Descarga del Repositorio Expuesto**

El ataque se realizó usando un script diseñado para reconstruir repositorios desde archivos `.git` públicos.

**Herramienta utilizada:** `git-dump.py`

Se realizaron adaptaciones al script:

* `urlparse` → `urllib.parse`
* `raw_input` → `input`
* Corrección de errores de sintaxis

**Comando utilizado:**

```bash
python3 git-dump.py http://127.0.0.1:1231/.git k8s-goat-git
```


#### **Análisis Forense del Repositorio**

Una vez clonado, el atacante tiene acceso al historial completo de commits.

Se usó **trufflehog** para detectar secretos mediante entropía y patrones.



###  **Hallazgo y Consecuencias Técnicas**

* **Secreto Comprometido:** trufflehog reveló credenciales sensibles y la **flag final (k8s_goat_flag)**.
* **Riesgo Principal:**

  * Exfiltración de código fuente
  * Robo de claves API que estaban hardcodeadas
  * Vulneración completa del historial del repositorio
<img width="949" height="1020" alt="Captura de pantalla 2025-11-27 190531" src="https://github.com/user-attachments/assets/fb2fcadc-02a1-4a07-a8b5-ab39083a622f" />
<img width="776" height="808" alt="Captura de pantalla 2025-11-27 202409" src="https://github.com/user-attachments/assets/45755e3d-f8ac-4c5c-aebf-092ff09ce573" />



# ## **2. DIND (Docker-in-Docker) Exploitation (Escenario 2)**

###  **Descripción de la Vulnerabilidad**

El servicio **Ping Your Servers** (Pod: `internal-proxy-deployment`) presentaba dos fallas críticas:

1. **Inyección de Comandos (Command Injection):**
   La entrada del usuario no estaba sanitizada, permitiendo ejecutar comandos arbitrarios.

2. **Montaje de Socket del Host:**
   El Pod tenía montado el socket del runtime del host:

   ```
   /custom/containerd/containerd.sock
   ```

Esto permite escalar de vulnerabilidad de aplicación → vulnerabilidad de infraestructura.


### 🛠️ **Detalle del Proceso de Explotación**

#### **1. Confirmación de Inyección de Comandos**

Se utilizó el separador `;` para ejecutar comandos adicionales.

Ejemplo:

```
127.0.0.1; id
```

Esto permitió identificar al usuario del contenedor y confirmar el RCE.



#### **2. Identificación del Socket Montado**

Mediante comandos inyectados como:

```
mount
```

Se descubrió el socket `containerd` del host.


#### **3. Explotación Docker-in-Docker**

Con el socket del host accesible, se usó `crictl` para interactuar directamente con el runtime del nodo.

**Comando utilizado:**

```bash
/tmp/crictl -r unix:///custom/containerd/containerd.sock images
```

Este comando permite ver **todas las imágenes del nodo anfitrión**, confirmando escalación de privilegios.


###  **Hallazgo y Consecuencias Técnicas**

#### **Riesgo Máximo: Toma de Control del Nodo Anfitrión**

El atacante tendría capacidad para:

* Detener Pods del clúster
* Extraer secretos de cualquier contenedor
* Crear contenedores privilegiados
* Manipular redes o volúmenes del host

Comprometiendo completamente el Worker Node.



# ## **NOTAS**

###  **Vulnerabilidad y Contexto**

Un Pod con el nombre `exec-into-pod-with-clusterrole-*` tenía asignado un **ClusterRoleBinding** con permisos excesivos:

* `resources: ["*"]`
* `verbs: ["exec"]`

Esto viola completamente el principio de mínimo privilegio.



**Impacto y Consecuencias Técnicas**

#### **Violación de RBAC (Access Control)**

Un Pod con permisos simples podía ejecutar comandos en *cualquier* Pod del clúster.

#### **Movimiento Lateral**

Si un atacante comprometiera este Pod, podría ejecutar:

```bash
./kubectl exec kubernetes-goat-home-deployment-76847b6f98-bz5dg -- cat /var/www/html/k8s-goat-flag.txt
```

Con esto, podría robar la flag del Pod principal.

Esto demuestra que RBAC mal configurado equivale a entregar control total del cluster.

<img width="962" height="920" alt="Captura de pantalla 2025-11-27 233937" src="https://github.com/user-attachments/assets/d94b61a8-64b8-4385-8e0f-5f32eceede0b" />


# ## Recomendaciones de Mitigación

| Vulnerabilidad        | Recomendación Técnica                                                                                                                  |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| **Sensitive Keys**    | Aplicar escaneo automático de secretos (Gitleaks, TruffleHog), bloquear `.git` desde HTTP, usar Git hooks preventivos.                 |
| **DIND Exploitation** | Sanitizar entradas de usuario, prohibir montajes de sockets del host en Pods no privilegiados, aplicar Pod Security Standards.         |
| **RBAC Excesivo**     | Auditar roles y permisos, evitar `resources: ["*"]`, aplicar mínimo privilegio estricto, usar Network Policies como defensa adicional. |



