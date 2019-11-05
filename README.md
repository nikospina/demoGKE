# Requisitos previos
Hay algunas cosas que debe hacer antes de comenzar de verdad:

Instalar Terraform
Instalar kubectl
Instale el SDK de Google Cloud
Clonar el repositorio complementario para esta publicación de blog

# Crear cuenta de servicio
Para Terraform, queremos credenciales del tipo de cuenta de servicio. Vamos a la consola de administración de IAM para hacer eso. Haga clic en Crear cuenta de servicio y complete el formulario con los siguientes datos:

    Service account name: terraform
    Service account ID: terraform
    Service account description: Desplegar infraestructura utilizando Terraform.
  
El siguiente paso es asignar permisos.

  Service account permissions:
  
    Administrador de Compute
    Administrador de Kubernetes Engine
    Usuario de la cuenta de servicio
    Administrador de almacenamiento
    
Finalmente, en el último paso, la parte más importante es crear una clave y descargarla en formato JSON.
Guárdelo en el lugar donde clonó este repositorio. Llamelo account.json.

# Crear un depósito en la seccion de storage y habilitar el control de versiones
Seguiremos la documentación oficial en Google Storage sobre cómo configurar un bucket. Desafortunadamente, la consola de Google Storage actualmente no puede permitirnos habilitar el control de versiones, que es algo que Terraform recomienda (requiere). Para eso, necesitamos descargar e instalar Google Cloud SDK , lo que nos da acceso a gsutil. Una vez que haya seguido las instrucciones para el SDK de Google Cloud, puede escribir los siguientes comandos:

    $ export BUCKET_ID=...your ID goes here...
    $ gsutil versioning set on gs://${BUCKET_ID}
        //Enabling versioning for gs://gke-from-scratch-terraform-state/...
    $ gsutil versioning get gs://${BUCKET_ID}
        //gs://gke-from-scratch-terraform-state: Enabled
  
# Configuración de Google Cloud Storage como estado remoto de Terraform
Configuraremos un espacio de trabajo de Terraform para cada una de nuestras implementaciones. Recuerde que podría haber una razón para que implementemos varias réplicas de este entorno (almacenamiento provisional, recuperación ante desastres, etc.). Definamos uno llamado "production". Hay un archivo en el repositorio adjunto llamado production.tfvars, que puede inspeccionar, modificar y usar.

La mayoría de los valores en los archivos de Terraform se han escrito de tal manera que solo necesita modificar el archivo production.tfvars. Sin embargo, un valor codificado está en google.tf, en la línea donde se especifica el depósito/bucket. La razón de esta desafortunada limitación es técnica: Terraform no permite variables en esa sección en particular. Así que cámbialo en tugoogle.tf archivo y luego proceder.

Para configurar Terraform, ejecutamos:
 
    $ export ENVIRONMENT=production
    $ terraform workspace new ${ENVIRONMENT}  
      //Created and switched to workspace "production"!  
      //You're now on a new, empty workspace. Workspaces isolate their state,
      //so if you run "terraform plan" Terraform will not see any existing state
      //for this configuration.
    $ terraform init -var-file=${ENVIRONMENT}.tfvars  
  
Terraform nos da muchos resultados, pero entre otros, dirá que el proveedor "google" se ha configurado y que el proveedor "gcs" (backend de estado remoto) se ha configurado. ¡Éxito!

# Usando Terraform para crear un clúster GKE
Cuando crea un clúster GKE, termina automáticamente con un grupo de nodos inicial . Si bien eso suena bien, el problema es que no podemos administrarlo y establecer la configuración que queremos (actualizaciones automáticas, reparación automática y escalado automático). Entonces le pediremos a Terraform que cree un clúster y luego elimine el grupo de nodos inicial. Luego se le pedirá que cree un nuevo grupo de nodos (que tenga configuradas las opciones de configuración) y luego lo adjunte al clúster. Esto es tan común (y el enfoque recomendado) que Terraform tiene una opción especial para ello.

# Darle a nuestra cuenta de servicio los permisos requeridos
Si intentáramos implementar ahora, nos enfrentaríamos a un error de permisos. Necesitamos dar más permisos a nuestra cuenta de servicio, y el papel de roles/editor. Para hacerlo, emita el siguiente comando:

     $ export PROJECT=gke-from-scratch
     $ export SERVICE_ACCOUNT=terraform
     $ gcloud projects add-iam-policy-binding ${PROJECT} --member serviceAccount:${SERVICE_ACCOUNT}@${PROJECT}.iam.gserviceaccount.com --role roles/editor
  
# Implementación del clúster GKE
Ahora que hemos hecho todos los preparativos, es casi vergonzosamente fácil implementar el clúster:

    $ terraform apply -var-file=${ENVIRONMENT}.tfvars

Responda "sí" cuando se le solicite, y después de unos minutos, ¡tiene un clúster! Puede dirigirse a la lista de Kubernetes Clusters en la consola de Google Cloud.
