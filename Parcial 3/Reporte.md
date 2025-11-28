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


<img width="962" height="920" alt="Captura de pantalla 2025-11-27 233937" src="https://github.com/user-attachments/assets/d94b61a8-64b8-4385-8e0f-5f32eceede0b" />


# ## Recomendaciones de Mitigación

| Vulnerabilidad        | Recomendación Técnica                                                                                                                  |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| **Sensitive Keys**    | Aplicar escaneo automático de secretos (Gitleaks, TruffleHog), bloquear `.git` desde HTTP, usar Git hooks preventivos.                 |
| **DIND Exploitation** | Sanitizar entradas de usuario, prohibir montajes de sockets del host en Pods no privilegiados, aplicar Pod Security Standards.         |
| **RBAC Excesivo**     | Auditar roles y permisos, evitar `resources: ["*"]`, aplicar mínimo privilegio estricto, usar Network Policies como defensa adicional. |


## 3. SSRF en el Mundo de Kubernetes (Escenario 3)

###  Descripción de la Vulnerabilidad
La aplicación expone una vulnerabilidad de **Server-Side Request Forgery (SSRF)**. Esta falla permite al atacante forzar a la aplicación a realizar peticiones a recursos internos a los que normalmente no tendría acceso directo. En Kubernetes, el objetivo principal es el **Metadata API Endpoint** [link: Scenario 3 - SSRF in the Kubernetes World].

###  Detalle del Proceso de Explotación
La explotación se centra en acceder al **token de la cuenta de servicio** del Pod, que se utiliza para la comunicación interna con el API Server de Kubernetes.

1.  **Identificación de la API:** Se utiliza la vulnerabilidad SSRF para realizar una solicitud al *endpoint* de la API de metadatos interna de Kubernetes, típicamente a través del *Service* de `kubernetes` (ejemplo: `http://kubernetes.default.svc`).
2.  **Exfiltración del Token:** La explotación se dirige a la ruta donde se monta el token de la Cuenta de Servicio (Service Account) dentro del Pod, que es `/var/run/secrets/kubernetes.io/serviceaccount/token`. El atacante fuerza a la aplicación a leer y devolver el contenido de este archivo sensible.
3.  **Uso del Token:** Una vez obtenido el token, el atacante puede usarlo con herramientas como `curl` o `kubectl` (fuera del Pod) para autenticarse en el **API Server** de Kubernetes y realizar acciones permitidas por el *Role* asignado al Pod.

###  Hallazgo y Consecuencias Técnicas
* **Vulnerabilidad:** Server-Side Request Forgery (SSRF).
* **Impacto:** **Compromiso del Control de Acceso (RBAC)**. El atacante roba la identidad del Pod (el token de Service Account). Si el Pod tiene permisos excesivos (como `get secrets` o `exec` en otros Pods), el atacante hereda esos permisos, logrando **Movimiento Lateral** y **Acceso a Secretos del Clúster**.


## 4. Container Escape al Host (Escenario 4)

###  Descripción de la Vulnerabilidad
El Pod vulnerable está mal configurado al permitir un **montaje de volumen inseguro** del directorio raíz del *host* (`/`) dentro del contenedor, típicamente montado como `/host` [link: Scenario 4 - Container Escape to the Host System]. Aunque el *socket* del *runtime* no está expuesto directamente (como en el Escenario 2), el acceso a los *filesystems* del *host* crea una ruta de escape.

### 🛠️ Detalle del Proceso de Explotación
El ataque utiliza el acceso al sistema de archivos del *host* para realizar acciones que fuerzan la ejecución de código fuera del contenedor.

1.  **Acceso al Host:** El atacante accede al *shell* del Pod.
2.  **Identificación del Montaje:** Se confirma el *mount* del *filesystem* raíz del *host* dentro del contenedor (ejemplo: `/host`).
3.  **Escalada:** La explotación se logra escribiendo comandos o claves SSH en directorios críticos del *host* a través del punto de montaje. Un método común es **modificar el archivo `/host/etc/crontab`** o **`/host/root/.ssh/authorized_keys`** para inyectar código malicioso que el *host* ejecutará.
    * *Ejemplo:* Inyectar una clave SSH pública en el archivo `authorized_keys` del *host* para establecer una conexión SSH inversa o remota.

###  Hallazgo y Consecuencias Técnicas
* **Vulnerabilidad:** Montaje Inseguro de `hostPath` (host's root filesystem).
* **Impacto:** **Escape Directo del Contenedor (Container Escape)**. Esta es una de las fallas más graves, ya que el atacante puede leer, escribir y modificar archivos en el sistema operativo del *host*, logrando **ejecución de código arbitrario** en la máquina física o virtual que aloja Kubernetes.


## 5. Docker CIS Benchmarks (Escenario 5)

###  Descripción de la Vulnerabilidad
El Pod de este escenario está configurado con **privilegios de *capabilities*** excesivos que violan las mejores prácticas de seguridad, como las definidas en el **Docker CIS Benchmark** [link: Scenario 5 - Docker CIS Benchmarks in Kubernetes Containers]. Específicamente, el Pod retiene la *capability* **`CAP_NET_RAW`**.

### 🛠️ Detalle del Proceso de Explotación
La explotación demuestra cómo una *capability* aparentemente inofensiva puede utilizarse para propósitos maliciosos fuera del objetivo principal de la aplicación.

1.  **Acceso y Enumeración:** El atacante obtiene un *shell* en el Pod y enumera las *capabilities* asignadas utilizando herramientas como `capsh` o inspeccionando el manifiesto del Pod. Se confirma la presencia de `CAP_NET_RAW`.
2.  **Ataque de Red:** La *capability* `CAP_NET_RAW` permite al contenedor crear *sockets* de red sin procesar (raw sockets). Esto habilita ataques de bajo nivel, como **spoofing de paquetes IP** o la ejecución de **ataques de *sniffing*** (captura de tráfico de red) que exponen la comunicación entre otros Pods y servicios dentro del *namespace* o la red del *host*.
3.  **Comando de Prueba:** Herramientas de red como `tcpdump` o `ping` avanzado se ejecutan para demostrar que el Pod puede manipular paquetes de red crudos en el entorno.

###  Hallazgo y Consecuencias Técnicas
* **Vulnerabilidad:** Asignación excesiva de *Capabilities* de Linux (`CAP_NET_RAW`).
* **Impacto:** **Ataques de Red de Bajo Nivel**. El contenedor puede evadir las protecciones normales de red, monitorear el tráfico sensible (contraseñas, tokens) de otros Pods en el mismo nodo (si se usa `hostNetwork`) o realizar ataques de suplantación, comprometiendo la privacidad de las comunicaciones internas.


###  Recomendaciones Generales

| Vulnerabilidad | Recomendación Técnica de Mitigación |
| :--- | :--- |
| **SSRF (Esc. 3)** | **Filtrado de Salida:** Implementar reglas de *firewall* (o *Network Policies*) para evitar que los Pods se comuniquen con **API de Metadatos del Nube** (ej. `169.254.169.254`) o la **API Interna de Kubernetes**. |
| **Container Escape (Esc. 4)** | **Restringir `hostPath`:** Prohibir el montaje de directorios sensibles del *host* (especialmente `/`) dentro de contenedores. Usar Pod Security Standards (PSS) para aplicar restricciones a `hostPath`. |
| **Docker CIS (Esc. 5)** | **Mínimo Privilegio de Capabilities:** Usar el perfil de seguridad de *runtime* (ej. `seccomp`) para descartar todas las *capabilities* innecesarias y solo conservar las esenciales (ej. `NET_BIND_SERVICE`). Evitar **`CAP_NET_RAW`**. |
| **RBAC Excesivo** | **Auditar Service Accounts y Roles:** Usar herramientas como `kube-hunter` o `Kubeaudit` para identificar roles con permisos demasiado amplios (`verbs: ["*"]`, `resources: ["*"]`). |

# ## **NOTAS**
1. Tener instalado el Docker Desktop
2. Configurar el Docker Desktop
   
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

