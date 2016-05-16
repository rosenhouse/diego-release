# MySQL Backend for Diego

These instructions allow you to either:

* Provision an RDS MySQL Instance as a backend
* Provision a stand-alone CF-MySQL release
* Configure Diego to use one of the above configurations

## Table of Contents

1. [Setup RDS MySQL](#setup-aws-rds-mysql)
1. [Deploy Standalone CF-MySQL](#deploy-standalone-cf-mysql)
1. [Deploying Diego](#deploy-diego)

## Setup AWS RDS MySQL
Support for using a SQL database instead of etcd for the backing store of Diego is still in the experimental phase. The instructions below describe how to set up a MariaDB RDS instance that is known to work with Diego.

1. From the AWS console homepage, click on `RDS` in the `Database` section.
1. Click on `Launch DB Instance` under Instances.
1. Click on the `MariaDB` Tab and click the `Select` button.
1. Select Production or Dev/Test version of MariaDB depending on your use case and click the `Next Step` button.
1. Select the DB Instance Class required. For performance testing the Diego team uses db.m4.4xlarge.
1. Optionally tune the other parameters based on your deployment requirements.
1. Provide a unique DB Instance Identifier. This identifier can be arbitrary, as is not used directly in the Diego configuration below.
1. Choose and confirm a master username and password, and record them for later use in the Diego-SQL stub.
1. Click `Next Step`.
1. Select the VPC created during the bosh-init steps above.
1. Select `No` for the `Publicly Accessible` option.
1. Select the `VPC Security Group` matching `*-InternalSecurityGroup-*`.
1. Choose a Database Name (for example, `diego`).
1. Click `Launch DB Instance`.
1. Wait for the Instance to be `available`.

### Configuring SSL

In order to configure SSL for RDS you need to download the ca cert bundle from AWS. This can be done by:

```bash
curl -o $DEPLOYMENT_DIR/certs/rds-combined-ca-bundle.pem http://s3.amazonaws.com/rds-downloads/rds-combined-ca-bundle.pem
```

The contents of this file will be supplied in the `sql_overrides.bbs.ca_cert` field in the Diego-SQL stub below.

## Deploy Standalone CF-MySQL

Follow the instructions at [CF MySQL Deploy](https://github.com/cloudfoundry/cf-mysql-release#deploy-on-aws-or-vsphere) to deploy a stand alone example [examples/standalone](https://github.com/cloudfoundry/cf-mysql-release/blob/develop/manifest-generation/examples/standalone)

**Note**: Currently the `master` branch of CF-MySQL (v26) cannot be checked out due to a missing submodule. We recommend that you use the `release-candidate` branch until a new final release is cut (v27).

You will need to make changes to the `instance-count-overrides.yml` and `property-overrides.yml` found [here](https://github.com/cloudfoundry/cf-mysql-release/tree/master/manifest-generation/examples/standalone),
as well as the `iaas-settings.yml` stub file found [here](https://github.com/cloudfoundry/cf-mysql-release/blob/master/manifest-generation/examples/aws/iaas-settings.yml).

To minimize the deployment to only a single MySQL node use the following `instance-count-overrides.yml`

```yaml
instance_count_overrides:
  - name: cf-mysql-broker_z1
    instances: 0
  - name: cf-mysql-broker_z2
    instances: 0
  - name: mysql_z2
    instances: 0
  - name: arbitrator_z3
    instances: 0
  - name: proxy_z1
    instances: 0
  - name: proxy_z2
    instances: 0
```

You will also need to seed the mysql deployment with a Diego database, username, and password. This can be done by providing the following property in your `property_overrides.yml`:

```yaml
property_overrides:
  mysql:
    seeded_databases: |
      - name: diego
        username: diego
        password: REPLACE_ME_WITH_DB_PASSWORD
```

When filling out the [`iaas_settings.yml`](https://github.com/cloudfoundry/cf-mysql-release/blob/develop/manifest-generation/examples/aws/iaas-settings.yml),
you only need to create one subnet and fill in the corresponding properties for the `mysql1` network/AZ.
You can create an AWS subnet for the CF-MySQL deployment by:

1. From the AWS console homepage, click on `VPC` in the `Networking` section.
1. Click on the `Subnets` link.
1. Click on the `Create Subnet` button.
1. Fill in the name tag property for the subnet.
1. Select the VPC associated with your deployment.
1. Select the preferred AZ for your CF-MySQL deployment.
1. Fill in `10.10.32.0/24` as the CIDR range.
1. Click on the `Yes, create` button.


## Deploy Diego

### Fill in Diego-SQL stub

To configure Diego to communicate with the SQL instance, first create a Diego-SQL stub file at `$DEPLOYMENT_DIR/stubs/diego/diego-sql.yml` with the following contents:

```yaml
sql_overrides:
  bbs:
    db_connection_string: 'diego:REPLACE_ME_WITH_DB_PASSWORD@tcp(<sql-instance-endpoint>)/diego'
    max_open_connections: 500
    require_ssl: true
    ca_cert: |
      REPLACE_WITH_CONTENTS_OF_(DEPLOYMENT_DIR/certs/rds-combined-ca-bundle.pem)
```

Fill in the bracketed parameters in the `db_connection_string` with the following values:

- `REPLACE_ME_WITH_DB_PASSWORD`: The password chosen when you created the SQL instance.
- `<sql-instance-endpoint>`:
	- For AWS RDS: The endpoint displayed at the top of the DB instance details page in AWS, including the port.
	- For Standalone CF-MySQL: The internal IP address and port of the single MySQL node. (e.g. 10.10.5.222:3306). By default the port will be 3306.

**Note:** The `sql_overrides.bbs.ca_cert` and `sql_overrides.bbs.require_ssl` properties should be provided only when deploying with an SSL-supported MySQL cluster. Set the `require_ssl` property to `true` to ensure that the BBS uses SSL to connect to the store, and set the `ca_cert` property to the contents of a certificate bundle containing the correct CA certificates to verify the certificate that the SQL server presents.

### Generate the Diego manifest

Generate the Diego manifest with an additional `-s` flag that specifies the location of the Diego-SQL stub, as shown below. Remember that the `-n` instance-count-overrides flag and the `-v` release-versions flags are optional.

```bash
cd $DIEGO_RELEASE_DIR
./scripts/generate-deployment-manifest \
  -c $DEPLOYMENT_DIR/deployments/cf.yml \
  -i $DEPLOYMENT_DIR/stubs/diego/iaas-settings.yml \
  -p $DEPLOYMENT_DIR/stubs/diego/property-overrides.yml \
  -s $DEPLOYMENT_DIR/stubs/diego/diego-sql.yml \
  -n $DEPLOYMENT_DIR/stubs/diego/instance-count-overrides.yml \
  -v $DEPLOYMENT_DIR/stubs/diego/release-versions.yml \
  > $DEPLOYMENT_DIR/deployments/diego.yml
```