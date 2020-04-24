+++
title = "Programmatic Firestore index creation in go"
description = "Programmatically create Firestore indexes in Go"
date = "2019-09-22"
tags = ["go", "firestore"]
+++

Two months ago I started building [Google Cloud, native storage](https://github.com/gravitational/teleport/pull/2821) support for cluster data and session recording
by [Gravitational's](https://gravitational.com) [Teleport](https://github.com/gravitational/teleport). The objective was to mirror
the S3/DynamoDB support for AWS, but for GCP. GCS is the obvious equivalent of S3, but replacing
DynamoDB was not so obvious... Prior to the project I hadn't actually used any Firebase tooling and I
had to wrap my head around which implementation of [Firestore to use](https://cloud.google.com/datastore/docs/firestore-or-datastore);
native mode or datastore mode.

Ultimately the decision came down to which of the two supported realtime updates, a feature mostly
meant for devices in hands, but also benefits server backends, particularly in cases where you don't
want to have to deal with additional messaging systems. Ultimately, then, the decision
came down to Firestore's native-mode watch support for real-time updates.

Firestore is designed to perform well and be as operator/dev friendly as possible with things like automatic
indexes on fields, etc... An example testament to Firestore's friendliess: if you try
and query across multiple fields without a compound index, Firestore will bark at you and produce an error
with a link to create the index. Normally, in a schema-driven system, you'd use a schema
tracking/changing/migration tool to ensure these indexes, but Firestore is schemaless and thus
the only real maintenance in keeping your data access patterns in check with changing needs.

Normally I'd use [Terraform](https://www.terraform.io) to manage these indexes, the
[`google_firestore_index`](https://www.terraform.io/docs/providers/google/r/firestore_index.html) resources, but
the Gravitational/Teleport team requires the app ensure its cloud resources exist prior to completing startup.
This meant I needed to code index creation into the Firestore events
and Firestore cluster/data storage backends. The effort spent to get there was more than it should have
been and I'm hoping this post makes that quicker and less painful for anyone else looking to do the same.

First up, imports... the bulk of Firestore operations will come from the package: `cloud.google.com/go/firestore`
but you won't find anything useful [therein](https://godoc.org/cloud.google.com/go/firestore) for index operations.
To programmatically create indexes you'll need to import a few additional packages:

- `apiv1 "cloud.google.com/go/firestore/apiv1/admin"` - [Firestore Admin API](https://godoc.org/cloud.google.com/go/firestore/apiv1/admin)
- `adminpb "google.golang.org/genproto/googleapis/firestore/admin/v1"` - [Firestore admin protobufs](https://godoc.org/google.golang.org/genproto/googleapis/firestore/v1)

Firestore admin clients are different, but similarly instantiated compared with the normal clients:

`firestoreAdminClient, _ := apiv1.NewFirestoreAdminClient(context.Background(), args...)`

_Ignore the part above where I do bad things like ignoring error outputs from functions._

Say, for example, I want to create a compound index on the collection `cheeses` on the field
`type` and `age` both in ascending order. (Note: the direction of the index is important and must
match the expected queries.) `cheeses` lives in the GCP project `jdurbin-cheeese-factory`.

The index parent, a string, is the fully qualified parent path of the collection.

`indexParent := fmt.Sprintf("projects/%s/databases/(default)/collectionGroups/%s", "jdurbin-cheese-factory", "cheeses")`

Next up, create the objects to pass to the admin client for index creation.

```go
# order
ascendingFieldOrder := adminpb.Index_IndexField_Order_{
  Order: adminpb.Index_IndexField_ASCENDING,
}

# fields
fields := make([]*adminpb.Index_IndexField, 0)
fields = append(fields, &adminpb.Index_IndexField{
  FieldPath: "type",
  ValueMode: &ascendingFieldOrder,
})
fields = append(fields, &adminpb.Index_IndexField{
  FieldPath: "age",
  ValueMode: &ascendingFieldOrder,
})
```

Moving on, pass the fields and index parent to the admin client... (note `ctx` is pre-defined)

```go
operation, err := adminSvc.CreateIndex(ctx, &adminpb.CreateIndexRequest{
  Parent: indexParent, # should be projects/jdurbin-cheese-factory/databases/(default)/collectionGroups/cheese
  Index: &adminpb.Index{
    QueryScope: adminpb.Index_COLLECTION,
    Fields:     fields, # aforementioned, created fields
  },
})

if err != nil && status.Convert(err).Code() != codes.AlreadyExists {
  log.Debug("non-already exists error, returning.")
  return status.Convert(err).Err()
}
if operation != nil {
  meta := adminpb.IndexOperationMetadata{}
  _ = meta.XXX_Unmarshal(operation.Metadata.Value)

  # indexName is the index created by the job
  indexName := meta.Index
}
```

Firestore indexes take at minimum about 60 seconds to provision/create. If you're looking to ensure
the indexes prior to starting your app you'll need to block until you get an okay from the long running
index operation. Good thing the admin client's `CreateIndex` function returns a long running operation pointer!

`func (c *FirestoreAdminClient) CreateIndex(ctx context.Context, req *adminpb.CreateIndexRequest, opts ...gax.CallOption) (*longrunningpb.Operation, error)`

Though the call to `CreateIndex` returns a `longrunningpb.Operation` pointer, I don't use it to confirm or verify the index
creation, instead I used additional calls to the Firestore admin client with timeouts to check on the status of the index...

```
index, _ := adminSvc.GetIndex(ctx, &adminpb.GetIndexRequest{Name: "cheeses"})
if index.State == adminpb.Index_READY {
  // break and discontinue blocking
}
```

For a more complete example, see the `EnsureIndexes` func in the [Firestore backend implementation](https://github.com/joshdurbin/teleport/blob/gcp_ha_support/lib/backend/firestore/firestorebk.go#L706) for Teleport or see the complete block that follows:

```go
package main

import (
	"context"
	"fmt"
	"time"

	apiv1 "cloud.google.com/go/firestore/apiv1/admin"
	"google.golang.org/api/option"
	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"

	log "github.com/sirupsen/logrus"
	adminpb "google.golang.org/genproto/googleapis/firestore/admin/v1"
)

const (
	timeInBetweenIndexCreationStatusChecks = time.Second * 10
)

func main() {

	endPoint := ""
	credentialsFile := "/path/to/creds"

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	var args []option.ClientOption

	if len(endPoint) != 0 {
		args = append(args, option.WithoutAuthentication(), option.WithEndpoint(endPoint), option.WithGRPCDialOption(grpc.WithInsecure()))
	} else if len(credentialsFile) != 0 {
		args = append(args, option.WithCredentialsFile(credentialsFile))
	}

	firestoreAdminClient, _ := apiv1.NewFirestoreAdminClient(ctx, args...)

	defer firestoreAdminClient.Close()
	tuples := make([]*indexTuple, 0)

	// names and locations
	tuples = append(tuples, &indexTuple{
		FirstField:  "name",
		SecondField: "location",
	})

	// names and employers
	tuples = append(tuples, &indexTuple{
		FirstField:  "name",
		SecondField: "employer",
	})

	indexParent := fmt.Sprintf("projects/%s/databases/(default)/collectionGroups/%s", "project-id", "collection-id")

	ensureIndexes(ctx, firestoreAdminClient, tuples, indexParent)
}

type indexTuple struct {
	FirstField  string
	SecondField string
}

func ensureIndexes(ctx context.Context, adminSvc *apiv1.FirestoreAdminClient, tuples []*indexTuple, indexParent string) error {
	ascendingFieldOrder := adminpb.Index_IndexField_Order_{
		Order: adminpb.Index_IndexField_ASCENDING,
	}

	tuplesToIndexNames := make(map[*indexTuple]string)
	// create the indexes
	for _, tuple := range tuples {
		fields := make([]*adminpb.Index_IndexField, 0)
		fields = append(fields, &adminpb.Index_IndexField{
			FieldPath: tuple.FirstField,
			ValueMode: &ascendingFieldOrder,
		})
		fields = append(fields, &adminpb.Index_IndexField{
			FieldPath: tuple.SecondField,
			ValueMode: &ascendingFieldOrder,
		})
		operation, err := adminSvc.CreateIndex(ctx, &adminpb.CreateIndexRequest{
			Parent: indexParent,
			Index: &adminpb.Index{
				QueryScope: adminpb.Index_COLLECTION,
				Fields:     fields,
			},
		})
		if err != nil && status.Convert(err).Code() != codes.AlreadyExists {
			log.Debug("non-already exists error, returning.")
			return status.Convert(err).Err()
		}
		if operation != nil {
			meta := adminpb.IndexOperationMetadata{}
			_ = meta.XXX_Unmarshal(operation.Metadata.Value)
			tuplesToIndexNames[tuple] = meta.Index
		}
	}

	// check for statuses and block
	for {
		if len(tuplesToIndexNames) == 0 {
			break
		}
		time.Sleep(timeInBetweenIndexCreationStatusChecks)
		for tuple, name := range tuplesToIndexNames {
			index, _ := adminSvc.GetIndex(ctx, &adminpb.GetIndexRequest{Name: name})
			log.Infof("Index for tuple %s-%s, %s, state is %s.", tuple.FirstField, tuple.SecondField, index.Name, index.State.String())
			if index.State == adminpb.Index_READY {
				delete(tuplesToIndexNames, tuple)
			}
		}
	}

	return nil
}
```
