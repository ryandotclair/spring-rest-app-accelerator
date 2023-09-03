# Example Passwordless Spring Boot REST API with Azure Database for PostgreSQL

This is a scaffolded Spring Boot REST API project, deployable to Azure Spring Apps Enterprise. It's state stored in Azure Postgres with Passwordless authentication (best practice).

## Create the infrastructure

Run these commands from this folder.

```bash
#!/bin/sh

echo "Setting env variables"

export AZ_RESOURCE_GROUP=spring-rest-postgres
export AZ_ASAE_APP=$AZ_RESOURCE_GROUP
export AZ_DATABASE_SERVER_NAME=srp-$(echo $RANDOM-$RANDOM)
export AZ_DATABASE_NAME=data
export AZ_LOCATION=eastus
export AZ_ASAE_SERVICE=$AZ_DATABASE_SERVER_NAME
export AZ_POSTGRESQL_USERNAME=springy
export AZ_POSTGRESQL_PASSWORD=$(openssl rand -base64 32)
echo "Save this admin (springy) password!!!"
echo $AZ_POSTGRESQL_PASSWORD

export SPRING_DATASOURCE_URL=jdbc:postgresql://$AZ_DATABASE_NAME.postgres.database.azure.com:5432/$AZ_DATABASE_NAME
export SPRING_DATASOURCE_USERNAME=my.service.pg
export MY_AZ_ID=$(az ad signed-in-user show --query id -o tsv)
export MY_AZ_MAIL=$(az ad signed-in-user show --query mail -o tsv)

az postgres flexible-server create --location $AZ_LOCATION \
    --resource-group $AZ_RESOURCE_GROUP \
    --name $AZ_DATABASE_SERVER_NAME \
    --admin-user $AZ_POSTGRESQL_USERNAME \
    --admin-password $AZ_POSTGRESQL_PASSWORD \
    --sku-name Standard_B1ms \
    --tier Burstable \
    --public-access ALL \
    --storage-size 128

az postgres flexible-server ad-admin create -u $MY_AZ_MAIL \
    -i $MY_AZ_ID \
    -s $AZ_DATABASE_SERVER_NAME

az spring app create --name $AZ_ASAE_APP --assign-endpoint true

az spring connection create postgres-flexible \
    --resource-group $AZ_RESOURCE_GROUP \
    --service $AZ_ASAE_SERVICE \
    --app $AZ_ASAE_APP \
    --deployment default \
    --tg $AZ_RESOURCE_GROUP \
    --server $AZ_DATABASE_SERVER_NAME \
    --database $AZ_DATABASE_NAME \
    --client-type springboot \
    --connection my.service.pg \
    --system-identity

az spring app deploy -n $AZ_ASAE_APP --source-path . --build-env BP_JVM_VERSION=17

az spring app show -n $AZ_ASAE_APP --query properties.url

```