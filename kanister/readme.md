## Kanister Demo 

## How to Install Kanister 

add kanister helm repository 
```
helm repo add kanister https://charts.kanister.io/
```

create kanister namespace
```
kubectl create namespace kanister 
```
deploy kanister using helm 
```
helm install myrelease --namespace kanister kanister/kanister-operator --set image.tag=0.67.0
```

Once we have kanister deployed we should now show the CustomResourceDefinitions, this will show actionsets, blueprints, profiles.

```
kubectl get customresourcedefinitions.apiextensions.k8s.io | grep "kanister"
```

## Deploy your Application 

In this demo we will use MySQL 

```
helm repo add bitnami https://charts.bitnami.com/bitnami
kubectl create namespace mysql-test 
helm install mysql-release bitnami/mysql --namespace mysql-test \
    --set auth.rootPassword='asd#45@mysqlEXAMPLE'

```

## Create a Profile 
This will give us some where to store our backups, this will be done using the KanCTL CLI tool 

Run the following but make sure you have defined your environment variables 

- $AWS_ACCESS_KEY_ID
- $AWS_SECRET_KEY
- $AWS_BUCKET

```
kanctl create profile s3compliant --access-key $AWS_ACCESS_KEY_ID --secret-key $AWS_SECRET_KEY --bucket $AWS_BUCKET --region us-east-2 --namespace mysql-test
```

confirm you now have a profile created 
```
kubectl get profile -n mysql-test
```
## Creating a Blueprint 

Kanister uses Blueprints to define these database-specific workflows and open-source Blueprints are available for several popular applications. It's also simple to customize existing Blueprints or add new ones.


```
kubectl create -f https://raw.githubusercontent.com/kanisterio/kanister/master/examples/stable/mysql/mysql-blueprint.yaml -n kanister
```

this blueprint is created in the Kanister namespace check with environment

```
kubectl get blueprint -n kanister 
```

## Adding some data to MySQL 

```
kubectl exec -ti $(kubectl get pods -n mysql-test --selector=app.kubernetes.io/instance=mysql-release -o=jsonpath='{.items[0].metadata.name}') -n mysql-test -- bash
```
 after the above command you should see a prompt similar to: 

 ```
 I have no name!@mysql-release-0:/$
 ```

 We now need to authenticate into mysql with the following command 

 ```
 mysql --user=root --password=asd#45@mysqlEXAMPLE
 ```

 We can now create our database and add some data with the following commands: 

 ```
 CREATE DATABASE test;
 USE test;
 INSERT INTO pets VALUES ('Puffball','Diane','hamster','f','1999-03-30',NULL);
 SELECT * FROM pets;
 exit 
 exit 
 ```

## Creating an Actionset 

We have no data in our database at this point but this is to show you what it will look like in your AWS Location Profile. 

first we need our profile name 
```
kubectl get profile -n mysql
```
To then create our actionset (backupjob) we need to take the profile name and add to the profile in the following command. 

```
kanctl create actionset --action backup --namespace kanister --blueprint mysql-blueprint --statefulset mysql-test/mysql-release --profile mysql-test/s3-profile-d5k5r --secrets mysql=mysql-test/mysql-release
```

List out those ActionSets 

```
kubectl --namespace kanister get actionsets.cr.kanister.io
```

You can confirm that you have data in your profile by checking your AWS S3 Bucket directly as well you can check your actionset with the following command (remember backup-7zvtv will be different in your environment and you can get this from the previous command)

```
kubectl describe actionset backup-7zvtvv --namespace kanister 
```

## Let's cause a failure 

Connect into your mysql pod as we did before using the following two blocks of code. 

```
 kubectl exec -ti $(kubectl get pods -n mysql-test --selector=app.kubernetes.io/instance=mysql-release -o=jsonpath='{.items[0].metadata.name}') -n mysql-test -- bash
```
 ```
 mysql --user=root --password=asd#45@mysqlEXAMPLE
 ```
The following command will drop the test database i.e the one we created previously simulating a failure

 ```
 SHOW DATABASES;
 DROP DATABASE test;
 SHOW DATABASES;
 exit 
 exit 
 ```

## Restore the Data 
In order for us to restore the data from our AWS S3 Profile we need to create another actionset but this time it is a restore actionset. Note again that your actionset will be named different to what I have below. 

```
kanctl --namespace kanister create actionset --action restore --from "backup-7zvtv"
```
Now we have created a new actionset 

```
kubectl get actionset -n kanister
```

We can dive deeper into this and make sure the actions have performed correctly with the following describe command 

```
 kubectl describe actionset -n kanister restore-backup-7zvtv-rhx5x
 ```

## Let's make sure everything is back to normal 

```
 kubectl exec -ti $(kubectl get pods -n mysql-test --selector=app.kubernetes.io/instance=mysql-release -o=jsonpath='{.items[0].metadata.name}') -n mysql-test -- bash
```
 ```
 mysql --user=root --password=asd#45@mysqlEXAMPLE
 ```

 ```
SHOW DATABASES;
USE test;
SHOW TABLES;
SELECT * FROM pets;
exit
exit
```

