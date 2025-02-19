# Migrating your VMs to containers 

**Note:** It is useful to create a workspace for managing the migrations plans and artifacts. To create such workspace in your cloud shell environment run the commands below:
``` bash
mkdir ~/m4a-petclinic
cd ~/m4a-petclinic
```

## Migrating your MySQL VM to container
Start by creating a folder for your MySQL migration artifacts:
``` bash
mkdir mysql
cd mysql
```

1. Before migrating a GCE VM using Migrate for Anthos and GKE you must turn off the VM. Turn it off by running the command:  
``` bash
gcloud compute instances stop petclinic-mysql --project $PROJECT_ID --zone $ZONE_ID
```

2. Create a migration plan using 'migctl' command line interface (this should take a couple of minutes to finish):
``` bash
migctl migration create petclinic-db-migration --source my-ce-src --vm-id petclinic-mysql --intent ImageAndData
```
**Note that we are using the ImageAndData intent to migrate the VM and also create a persistent volume for the database data folder**

**Note:** You can check the migration plan generation using the command:
``` bash
migctl migration status petclinic-db-migration
```

3. Download and review the migration plan by running the command:
``` bash
migctl migration get petclinic-db-migration
```
It will create a file on your local file system called **petclinic-db-migration.yaml**. You can now open it in cloud shell editor by running the command `edit petclinic-db-migration.yaml`.

4. Modify petclinic-db-migration.yaml to create a persistent volume for the database data folder. You do so by finding the *- \<folder\>* section in the yaml file:   
``` yaml
  dataVolumes:

  # Folders to include in the data volume, e.g. "/var/lib/mysql"
  # Included folders contain data and state, and therefore are automatically excluded from a generated container image
  # Replace the placeholder with the relevant path and add more items if needed
  - folders:
      - <folder>
    pvc:
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
          # Modify the required disk size on the storage field below based on the capacity needed
            storage: 10G
```

You then need to replace `- <folder>` with `- /var/lib/mysql` and save the file. Your modified file should look like below:
<pre>
  dataVolumes:

  # Folders to include in the data volume, e.g. "/var/lib/mysql"
  # Included folders contain data and state, and therefore are automatically excluded from a generated container image
  # Replace the placeholder with the relevant path and add more items if needed
  - folders:
      <b>- /var/lib/mysql</b>
    pvc:
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
          # Modify the required disk size on the storage field below based on the capacity needed
            storage: 10G
</pre>

4. To update the migration plan with the folder modification you need to upload the modified yaml file. You do so by running the command:
``` bash
migctl migration update petclinic-db-migration --file petclinic-db-migration.yaml
```

5. Once you are satisfied with the migration plan, you may start generating the migration artifacts. You do so by running the command:
``` bash
migctl migration generate-artifacts petclinic-db-migration
```
**Note:** Artifacts generation duration may vary depending on the size of the source VM disk. You can check the progress of your migration by running the command:
``` bash
migctl migration status petclinic-db-migration
```
For a more verbose output you add the flag `-v` to the command above. 

6. When the artifacts generation is finished. You can download the generated artifacts using the get-artifacts command:
``` bash
migctl migration get-artifacts petclinic-db-migration
```
The downloaded artifacts includes the following files:
* **Dockerfile** - The Dockerfile is used to build the container image from your VM and make it executable by leveraging the Migrate for Anthos and GKE container runtime.
* **deployment_spec.yaml** - The deployment_spec.yaml file contains a working Kubernetes [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) a matching [Service](https://kubernetes.io/docs/concepts/services-networking/service/) which are used to deploy your newly migrated MySQL container and expose it via a service. It also includes a [PersistentVolume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) and a [PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#reserving-a-persistentvolume) for the database data folder.
* **blocklist.yaml** - The blocklist.yaml file allows you to enable/disable services that were discovered during the migration discovery phase.

7. One change that we would make to keep the service name consistent with the one expected by the Tomcat instance is to change the service name from `petclinic-mysql-mysqld` to `petclinic-mysql` in the `deployment_spec.yaml` file. First run the command:
``` bash
edit deployment_spec.yaml
```
Then find the kubernetes service named `petclinic-mysql-mysqld` and remove the trailing `-mysqld` from the name of the service. Save the file and you are now ready to deploy your MySQL container.

## Migrating your Tomcat VM to container
Start by creating a folder for your Tomcat migration artifacts:
``` bash
mkdir ~/m4a-petclinic/tomcat
cd ~/m4a-petclinic/tomcat
```
1. Before migrating a GCE VM using Migrate for Anthos and GKE you must turn off the VM. Turn it off by running the command:  
``` bash
gcloud compute instances stop tomcat-petclinic --project $PROJECT_ID --zone $ZONE_ID
```

2. Create a migration plan using 'migctl' command line interface. This time you will use the `Image` intent and the `tomcat` workload-type which will only migrate the Tomcat configuration and the applications deployed on it:
``` bash
migctl migration create petclinic-migration --source my-ce-src --vm-id tomcat-petclinic --intent Image --workload-type tomcat
```

**Note:** You can check the migration plan generation using the command:
``` bash
migctl migration status petclinic-migration
```

3. Download and review the migration plan by running the command:
``` bash
migctl migration get petclinic-migration
```
It will create a file on your local file system called **petclinic-migration.yaml**. You can now open it in cloud shell editor by running the command `edit petclinic-migration.yaml`. The migration plan should look like below:
``` yaml
tomcatServers:
  - # External paths required for running the Tomcat server or apps.
    additionalFiles: []
    # Edit this list of application paths to define migrated applications.
    applications:
      - /opt/tomcat/webapps/ROOT
      - /opt/tomcat/webapps/docs
      - /opt/tomcat/webapps/examples
      - /opt/tomcat/webapps/host-manager
      - /opt/tomcat/webapps/manager
      - /opt/tomcat/webapps/petclinic
      - /opt/tomcat/webapps/petclinic.war
    catalinaBase: /opt/tomcat
    catalinaHome: /opt/tomcat
    # Parent image for the generated container image.
    fromImage: tomcat:8.5.65-jdk16-openjdk
    imageName: tomcat-tomcat
    # Log Configuration paths for the Tomcat apps.
    logConfigPaths: []
    name: tomcat
    ports:
      - 8080
```

4. You can now change your migration plan to only migrate the petclinic application by removing ROOT, docs, examples, host-manager and manager applications from the `applications` list. You modified migration plan should now look like this:
``` yaml
tomcatServers:
  - # External paths required for running the Tomcat server or apps.
    additionalFiles: []
    # Edit this list of application paths to define migrated applications.
    applications:
      - /opt/tomcat/webapps/petclinic.war
    catalinaBase: /opt/tomcat
    catalinaHome: /opt/tomcat
    # Parent image for the generated container image.
    fromImage: tomcat:8.5.65-jdk16-openjdk
    imageName: tomcat-tomcat
    # Log Configuration paths for the Tomcat apps.
    logConfigPaths: []
    name: tomcat
    ports:
      - 8080
```

5. You need to update the migration plan by uploading the modified yaml file. You do so by running the command:
``` bash
migctl migration update petclinic-migration --file petclinic-migration.yaml
```

5. Once you are satisfied with the migration plan, you may start generating the migration artifacts. You do so by running the command:
``` bash
migctl migration generate-artifacts petclinic-migration
```
**Note:** Artifacts generation duration may vary depending on the size of the source VM disk. You can check the progress of your migration by running the command:
``` bash
migctl migration status petclinic-migration
```
For a more verbose output you add the flag `-v` to the command above. 

6. When the artifacts generation is finished. You can download the generated artifacts using the get-artifacts command below:
``` bash
migctl migration get-artifacts petclinic-migration
```
The downloaded artifacts will be downloaded into `tomcat` directory and include the following files:
* **Dockerfile** - The Dockerfile is used to build the container image for your Tomcat applications by leveraging the community Tomcat Docker image. The default image may be changed during migration by modifying `fromImage` or by modifying the `FROM` string in the Dockerfile.
* **build.sh** - The build.sh script is used to build the container image for your Tomcat by leveraging [Cloud Build](https://cloud.google.com/build). You **MUST** replace *<my_project>* with the project id which is going to store the image. Once changed, you can build your container image by running the command
``` bash
bash build.sh
```
* **deployment_spec.yaml** - The deployment_spec.yaml file contains a Kubernetes [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) and a matching [Service](https://kubernetes.io/docs/concepts/services-networking/service/) which are used to deploy your newly migrated Tomcat container and expose it via a service. **Note** that the service will be exposed via **ClusterIP** by default and you may change this to a LoadBalancer in-order to make your Tomcat application available externally. Before applying the deployment yaml you **MUST** replace *<my_project>* with the project id that was used in your `build.sh` script.
* **additionalFiles.tar.gz** - An archive containing any additional files that are needed by Tomcat and were specified in the migration plan.
* **applications.tar.gz** - An archive containing all the applications that were selected to be migrated in the migration plan.
* **catalinaHome.tar.gz** - An archive containing the original CATALINA_HOME content and it is used in the Dockerfile to override the default Tomcat settings.
* **logConfigs.tar.gz** - An archive containing log files (log4j2, log4j and logback) that were modified from logging to local filesystem to log to a console appender.

If you would like to expose your Tomcat via a load balancer you can modify the service definition in `deployment_spec.yaml` by adding the LoadBalancer type to the service:
<pre>
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    migrate-for-anthos-optimization: "true"
    migrate-for-anthos-version: v1.9
  name: tomcat
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  <b>type: LoadBalancer</b>
  selector:
    app: tomcat
status:
  loadBalancer: {}
</pre>

You are now ready to [deploy](../4-deploy/README.md) your migrated containers