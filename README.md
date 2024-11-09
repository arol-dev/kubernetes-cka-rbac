# Control de Acceso Basado en Roles (RBAC) Lab - CKA Course

Este laboratorio práctico está diseñado para aprender a gestionar el acceso a los recursos de Kubernetes mediante el Control de Acceso Basado en Roles (RBAC). RBAC es una característica fundamental en Kubernetes que permite a los administradores definir y limitar las acciones que los usuarios o grupos pueden realizar en el cluster. A través de este laboratorio, comprenderás cómo RBAC utiliza roles y asociaciones para otorgar permisos específicos a los usuarios.

Configurar el acceso para un usuario del equipo *developer* en un cluster Kubernetes, de manera que pueda listar los Pods en el namespace `default` utilizando permisos específicos. Este laboratorio explora el uso de certificados X.509 y la configuración de roles y asociaciones en Kubernetes.

La gestión de acceso mediante RBAC es esencial para el examen CKA, dado que se espera que los administradores configuren correctamente los permisos de los usuarios y gestionen la seguridad del cluster. Este laboratorio refuerza los conocimientos prácticos necesarios para el examen y también para escenarios reales de administración en Kubernetes.

Asegúrate de seguir cada paso cuidadosamente y practicar varias veces para estar completamente preparado para el examen.

## Prerrequisitos

- **VirtualBox** (se necesita **Python** y **pywin32** como prerrequisitos).
- **Vagrant**.
- **MobaXterm** para sesiones SSH.

## Objetivos

1. **Comprender los elementos básicos de RBAC**: roles, bindings, permisos y namespaces.
2. **Configurar certificados X.509** para la autenticación del usuario operador.
3. **Crear y asociar roles específicos** para otorgar permisos limitados sobre los recursos.
4. **Practicar el flujo de trabajo de configuración** de permisos de acceso, verificación de permisos y modificación de roles.

## Contenido del Repositorio

Este repositorio incluye:

- Una carpeta `scripts` con dos scripts que proporcionan soporte durante el laboratorio.
- Un fichero `Vagrantfile` que permite automatizar el despliegue de tres VMs en VirtualBox.

Las VMs consisten en:

- 1 nodo master.
- 2 nodos worker.

### Paso 1: Despliegue de las VMs

1. Clona el repositorio en tu entorno local:

   ```bash
   git clone https://github.com/arol-dev/kubernetes-cka-rbac.git
   cd kubernetes-cka-backup-recovery-rbac
   ```

2. Dentro del repositorio, ejecuta el siguiente comando para desplegar las VMs:

   ```bash
   vagrant up
   ```

   Esto comenzará a desplegar tres VMs en VirtualBox: un nodo master y dos worker nodes. Espera unos minutos para que el proceso termine.

3. Verifica el estado de las VMs con:

   ```bash
   vagrant status
   ```

   Asegúrate de que las tres máquinas estén en estado `running`.

4. Obtén la configuración SSH para conectarte a las máquinas:

   ```bash
   vagrant ssh-config
   ```

   Guarda los detalles proporcionados, ya que los necesitarás en el siguiente paso.

### Paso 2: Conectar a las VMs con MobaXterm

1. Abre **MobaXterm** y utiliza la configuración SSH obtenida anteriormente para conectarte a las tres máquinas.
   - No se requiere un usuario específico, deja el campo vacío.
   - Si se te solicita usuario o contraseña, utiliza la cadena `vagrant`.

### Paso 3: Configuración Inicial

1. **Crear directorio de trabajo**:
   ```bash
   cd /home/vagrant
   mkdir rbac
   ```

2. **Generar la llave para el certificado**:
   ```bash
   openssl genrsa -out rbac/operador.key 2048
   ```

3. **Crear la solicitud de firma del certificado (CSR)**:
   ```bash
   openssl req -new -key rbac/operador.key -out rbac/operador.csr -subj "/CN=operador/O=developer"
   ```

4. **Firmar el certificado con la Autoridad Certificadora del cluster**:
   ```bash
   sudo openssl x509 -req -days 365 -in rbac/operador.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out rbac/operador.crt
   ```

### Paso 4: Configuración de `kubeconfig`

1. **Añadir la información del cluster**:
   ```bash
   kubectl --kubeconfig=rbac/operador-config config set-cluster kubernetes --server=https://10.0.0.10:6443 --certificate-authority=/etc/kubernetes/pki/ca.crt --embed-certs=true
   ```

2. **Añadir el usuario con sus certificados**:
   ```bash
   sudo kubectl --kubeconfig=rbac/operador-config config set-credentials operador --client-certificate=rbac/operador.crt --client-key=rbac/operador.key --embed-certs=true
   ```

3. **Crear el contexto**:
   ```bash
   kubectl --kubeconfig=rbac/operador-config config set-context operador@kubernetes --user=operador --cluster=kubernetes --namespace=default
   ```

4. **Establecer el contexto como predeterminado**:
   ```bash
   kubectl --kubeconfig=rbac/operador-config config use-context operador@kubernetes
   ```

5. **Verificar el archivo `kubeconfig`**:
   ```bash
   kubectl --kubeconfig=rbac/operador-config config view
   ```

### Paso 5: Comprobación de permisos sin configurar el Role

1. **Configurar la variable `KUBECONFIG` para utilizar el nuevo archivo**:
   ```bash
   export KUBECONFIG=$PWD/rbac/operador-config
   ```

2. **Intentar listar los Pods (esperar un error)**:
   ```bash
   kubectl get pods
   ```

3. **Eliminar la variable `KUBECONFIG`**:
   ```bash
   unset KUBECONFIG
   ```

---

## Configuración de Roles y RoleBindings

### Paso 1: Crear el Role para listar Pods

1. **Crear el Role `developers` con permisos para listar Pods**:
   ```bash
   kubectl create role developers --resource=pods --verb=list
   ```

2. **Verificar el Role creado**:
   ```bash
   kubectl get roles
   kubectl describe role developers
   ```

### Paso 2: Crear el RoleBinding

1. **Crear el RoleBinding en YAML de forma declarativa :**
   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     name: developer-group
     labels:
       app: rbac
     namespace: default
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: Role
     name: developers
   subjects:
   - apiGroup: rbac.authorization.k8s.io
     kind: Group
     name: developer
   ```
   ```bash
   kubectl apply -f <nameyamlfile>
   ```

2. **Crear el RoleBinding de forma imperativa**:
   ```bash
   kubectl create rolebinding developer-group --group=developer --role=developers
   ```

### Paso 3: Verificación de permisos

1. **Configurar la variable `KUBECONFIG`**:
   ```bash
   export KUBECONFIG=$PWD/rbac/operador-config
   ```

2. **Listar los Pods en el namespace `default`**:
   ```bash
   kubectl get pods
   ```

3. **Intentar listar Pods en el namespace `kube-system` (esperar un error)**:
   ```bash
   kubectl get pods --namespace kube-system
   ```

4. **Probar el comportamiento de RBAC**:
   ```bash
   kubectl --user=operador get pods
   kubectl get events --sort-by=.metadata.creationTimestamp
   kubectl auth can-i --list
   ```

5. **Eliminar la variable `KUBECONFIG`**:
   ```bash
   unset KUBECONFIG
   ```

---

## Modificación del Role en Caliente

1. **Editar el Role `developers` para añadir permisos adicionales (listar, ver logs y ejecutar comandos)**:
   ```yaml
   rules:
   - apiGroups: [""]
     resources: ["pods", "pods/exec", "pods/log"]
     verbs: ["list", "get", "watch", "update", "create", "patch"]
   ```

2. **Actualizar permisos para aplicar a todos los recursos excepto el verbo `delete`**:
   ```yaml
   rules:
   - apiGroups: [""]
     resources: ["*"]
     verbs: ["list", "get", "watch", "update", "create", "patch"]
   ```

3. **Verificar los permisos del grupo `developers`**:
   ```bash
   export KUBECONFIG=$PWD/rbac/operador-config
   ```
---

## Finalización del Laboratorio

1. **Eliminar la variable `KUBECONFIG`**:
   ```bash
   unset KUBECONFIG
   ```

2. **Verificar los Roles y RoleBindings**:
   ```bash
   kubectl get role
   kubectl get rolebinding
   ```