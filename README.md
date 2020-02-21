This program generates code like this between your generated gqlgen and sqlboiler with support for Relay.dev (unique id's etc). This in work in progress and we are working on automatically generating the basis Mutations like create, update, delete working based on your graphql scheme and your database models.

To make this program a success tight coupling (same naming) between your database and graphql scheme is needed at the moment. The advantage of this program is the most when you have a database already designed. However everything is created with support for change so you could write some extra GrapQL resolvers if you'd like.

## Flow

1. Generate database structs with: https://github.com/volatiletech/sqlboiler  
   e.g. `sqlboiler mysql`
2. Generate GrapQL scheme from sqlboiler structs: https://github.com/web-ridge/sqlboiler-graphql-schema  
   e.g. `go run github.com/web-ridge/sqlboiler-graphql-schema --output=../schema.graphql`
3. Generate GrapQL structs with: https://github.com/99designs/gqlgen  
   e.g. `go run github.com/99designs/gqlgen`
4. Generate converts between gqlgen-sqlboiler with this program  
   e.g. `go run convert_plugin.go`

DONE: Generate converts between sqlboiler structs and graphql (with relations included)  
DONE: Generate converts between input models and sqlboiler  
DONE: Fetch sqlboiler preloads from graphql context  
DONE: Support for foreign keys named differently than their corresponding model
TODO: New plugin which generates CRUD resolvers based on mutations in graphql scheme  
TODO: Generate code which implements the generated where and search filters
TODO: Support CRUD of relationships inside input types
TODO: Support gqlgen multiple .graphql files

## Case

You have a personal project with a very big database and a 'Laravel API'. I want to be able to generate a new Golang GraphQL API for this project in no time.

## Example result of this plugin

```golang
func AddressToGraphQL(m *models.Address, root interface{}) *graphql_models.Address {
	if m == nil {
		return nil
	}
	r := &graphql_models.Address{
		ID:          AddressIDUnique(m.ID),
		Street:      helper.NullDotStringToPointerString(m.Street),
		HouseNumber: helper.NullDotStringToPointerString(m.HouseNumber),
		ZipAddress:  helper.NullDotStringToPointerString(m.ZipAddress),
		City:        helper.NullDotStringToPointerString(m.City),
		Longitude:   helper.TypesNullDecimalToFloat64(m.Longitude),
		Latitude:    helper.TypesNullDecimalToFloat64(m.Latitude),
		Description: helper.NullDotStringToPointerString(m.Description),
		Name:        helper.NullDotStringToPointerString(m.Name),
		Permission:  helper.NullDotBoolToPointerBool(m.Permission),
		DeletedAt:   helper.NullDotTimeToPointerInt(m.DeletedAt),
		UpdatedAt:   helper.NullDotTimeToPointerInt(m.UpdatedAt),
		CreatedAt:   helper.NullDotTimeToPointerInt(m.CreatedAt),
	}

	if !helper.UintIsZero(m.AddressStatusID) {
		if m.R != nil && m.R.AddressStatus != nil {
			rootValue, sameStructAsRoot := root.(*models.AddressStatus)
			if !sameStructAsRoot || rootValue != m.R.AddressStatus {
				r.AddressStatus = AddressStatusToGraphQL(m.R.AddressStatus, m)
			}
		} else {
			r.AddressStatus = AddressStatusWithUintID(m.AddressStatusID)
		}
	}

	if !helper.NullDotUintIsZero(m.CompanyID) {
		if m.R != nil && m.R.Company != nil {
			rootValue, sameStructAsRoot := root.(*models.Company)
			if !sameStructAsRoot || rootValue != m.R.Company {
				r.Company = CompanyToGraphQL(m.R.Company, m)
			}
		} else {
			r.Company = CompanyWithNullDotUintID(m.CompanyID)
		}
	}

	if !helper.NullDotUintIsZero(m.ContactPersonID) {
		if m.R != nil && m.R.ContactPerson != nil {
			rootValue, sameStructAsRoot := root.(*models.Person)
			if !sameStructAsRoot || rootValue != m.R.ContactPerson {
				r.ContactPerson = PersonToGraphQL(m.R.ContactPerson, m)
			}
		} else {
			r.ContactPerson = PersonWithNullDotUintID(m.ContactPersonID)
		}
	}

	if !helper.NullDotUintIsZero(m.HouseTypeID) {
		if m.R != nil && m.R.HouseType != nil {
			rootValue, sameStructAsRoot := root.(*models.HouseType)
			if !sameStructAsRoot || rootValue != m.R.HouseType {
				r.HouseType = HouseTypeToGraphQL(m.R.HouseType, m)
			}
		} else {
			r.HouseType = HouseTypeWithNullDotUintID(m.HouseTypeID)
		}
	}

	if !helper.NullDotUintIsZero(m.OwnerID) {
		if m.R != nil && m.R.Owner != nil {
			rootValue, sameStructAsRoot := root.(*models.Person)
			if !sameStructAsRoot || rootValue != m.R.Owner {
				r.Owner = PersonToGraphQL(m.R.Owner, m)
			}
		} else {
			r.Owner = PersonWithNullDotUintID(m.OwnerID)
		}
	}

	if !helper.UintIsZero(m.UserOrganizationID) {
		if m.R != nil && m.R.UserOrganization != nil {
			rootValue, sameStructAsRoot := root.(*models.UserOrganization)
			if !sameStructAsRoot || rootValue != m.R.UserOrganization {
				r.UserOrganization = UserOrganizationToGraphQL(m.R.UserOrganization, m)
			}
		} else {
			r.UserOrganization = UserOrganizationWithUintID(m.UserOrganizationID)
		}
	}
	if m.R != nil && m.R.Calamities != nil {
		r.Calamities = CalamitiesToGraphQL(m.R.Calamities, m)
	}
	if m.R != nil && m.R.People != nil {
		r.People = PeopleToGraphQL(m.R.People, m)
	}

	return r
}
```

sqlboiler.yml

```yaml
mysql:
  dbname: dbname
  host: localhost
  port: 8889
  user: root
  pass: root
  sslmode: "false"
  blacklist:
    - notifications
    - jobs
    - password_resets
    - migrations
mysqldump:
  column-statistics: 0
```

gqlgen.yml

```yaml
schema:
  - schema.graphql
exec:
  filename: graphql_models/generated.go
  package: graphql_models
model:
  filename: graphql_models/genereated_models.go
  package: graphql_models
resolver:
  filename: resolver.go
  type: Resolver
```

Run normal generator
`go run github.com/99designs/gqlgen -v`

Put this in your program convert_plugin.go e.g.

```golang
// +build ignore

package main

import (
	"fmt"
	"os"

	"github.com/99designs/gqlgen/api"
	"github.com/99designs/gqlgen/codegen/config"
	cm "github.com/web-ridge/gqlgen-sqlboiler/convert"
)

func main() {
	cfg, err := config.LoadConfigFromDefaultLocations()
	if err != nil {
		fmt.Fprintln(os.Stderr, "failed to load config", err.Error())
		os.Exit(2)
	}

	err = api.Generate(cfg,
		api.AddPlugin(cm.New(
			"helpers",        // directory where convert.go, convert_input.go and preload.go should live
			"models",         // directory where sqlboiler files are put
			"graphql_models", // directory where gqlgen models live
		)),
	)
	if err != nil {
		fmt.Println("error!!")
		fmt.Fprintln(os.Stderr, err.Error())
		os.Exit(3)
	}
}

```

`go run convert_plugin.go`
