# Terraform Redshift Provider
![Build Status](https://github.com/yurymkomarov/terraform-provider-redshift/workflows/Go/badge.svg?branch=master)

Manage Redshift users, groups, privileges, databases and schemas. It runs the SQL queries necessary to manage these (CREATE USER, DELETE DATABASE etc)
in transactions, and also reads the state from the tables that store this state, eg pg_user_info, pg_group etc. The underlying tables are more or less equivalent to the postgres tables, 
but some tables are not accessible in Redshift. 

Currently supports users, groups, schemas and databases. You can set privileges for groups on schemas. Per user schema privileges will be added at a later date. 

Note that schemas are the lowest level of granularity here, tables should be created by some other tool, for instance flyway. 

# Get it:
- Download for amd64 (for other architectures and OSes you can build from source as descibed below) from [release page](https://github.com/yurymkomarov/terraform-provider-redshift/releases)
- Add to terraform plugins directory: https://www.terraform.io/docs/configuration/providers.html#third-party-plugins

## Examples: 

Provider configuration
```
provider redshift {
  url      = "localhost"
  user     = "username"
  password = "password"
  database = "database"
}
```

Creating an admin user who is in a group and who owns a new database, with a password that expires
```
# Create a user
resource "redshift_user" "user" {
  username         = "username"
  password         = "password"
  valid_until      = "2018-10-30"
  connection_limit = 4
  createdb         = true
  syslog_access    = "UNRESTRICTED"
  superuser        = true
}

# Add the user to a new group
resource "redshift_group" "group" {
  group_name = "group"
  users      = [redshift_user.user.id]
}

# Create a schema
resource "redshift_schema" "schema" {
  schema_name       = "schema"
  owner             = redshift_user.user.id
  cascade_on_delete = true
}

# Give that group select, insert and references privileges on that schema
resource "redshift_group_schema_privilege" "privileges" {
  schema_id  = redshift_schema.schema.id
  group_id   = redshift_group.group.id
  select     = true
  insert     = true
  update     = false
  references = true
  delete     = false
}
```

Creating a user who can only connect using IAM Credentials as described [here](https://docs.aws.amazon.com/redshift/latest/mgmt/generating-user-credentials.html)

```
resource "redshift_user" "user" {
  username          = "username"
  password_disabled = true
  connection_limit  = 1
}
```

## Things to note
### Limitations
For authoritative limitations, please see the Redshift documentations. 
1) You cannot delete the database you are currently connected to. 
2) You cannot set table specific privileges since this provider is table agnostic (for now, if you think it would be feasible to manage tables let me know)
3) On importing a user, it is impossible to read the password (or even the md hash of the password, since Redshift restricts access to pg_shadow)

### Prequisites to development
1. Go installed
2. Terraform installed locally

### Building: 
1. Run `go build -o terraform-provider-redshift`. You will need to tweak this with GOOS and GOARCH if you are planning to build it for different OSes and architectures
2. Add to terraform plugins directory: https://www.terraform.io/docs/configuration/providers.html#third-party-plugins

You can debug crudely by setting the TF_LOG env variable to DEBUG:
```
$ TF_LOG=DEBUG terraform apply
```

## ToDo
1. Database property for Schema
2. Schema privileges on a per user basis
3. Add privileges for languages and functions
