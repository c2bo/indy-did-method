## DID Operations

### Creation

Creation of a `did:indy` DID is performed by an authorized entity executing a `NYM` ledger transaction on a given Indy network. An Indy [[ref: NYM]] transaction includes an identifier (`dest`), an ED25519 verification key (`verkey`) and an optional JSON item (`diddocContent`). The [[ref: NYM]] is written to a specific Indy network with a given `namespace`. The following validation MUST be performed prior to executing the transaction to create the DID:

- Based on the configured authorization rules of the specific Indy ledger, the transaction may have to be signed by others, such as a Trustee or Endorser. If transaction is not authorized, the transaction MUST be rejected and an error returned to the client.

- A `did:indy` DID MUST be self-certifying by having the namespace identifier component of the DID (last element) derived from the initial public key of the DID, as follows:

    - For an Ed25519 key: Convert into Base58char the first 16 bytes of the 256 bit public key (verkey).

- The Indy ledger MUST verify the relationship between the namespace identifier component of the DID and the initial public key (verkey). If the relationship between the data elements fails verification, the transaction MUST be rejected and an error returned to the client.

- The [[ref: NYM]] transaction requires that the transaction to be written is signed by the DID controller. The ledger MUST verify the signature using the [[ref: NYM]] `verkey`. If the signature can not be validated, the transaction MUST be rejected and an error returned to the client.

- The Indy ledger MUST check that the data in the [[ref: NYM]] produces valid JSON and MUST do a limited DIDDoc validation check prior to writing the [[ref: NYM]] object to the ledger. Details of the assembly and verification are [below](#DIDDoc-Assembly-and-Verification). If the DIDDoc validation fails, the transaction MUST be rejected and an error returned to the client.

Although the DIDDoc is returned from the DIDDoc assembly and verification process, the DIDDoc is not used further by the ledger.

Once the validation checks are completed, the [[ref: NYM]] transaction is written to the Indy distributed ledger. If the [[ref: NYM]] write operation fails, an error is returned to the client.

On successfully writing the transaction to the Indy distributed ledger a success status is returned to the client.

#### DIDDoc Assembly and Verification

The DIDDoc returned when a `did:indy` DID is resolved is not directly stored in an Indy ledger document. Instead, the DIDDoc must be assembled from data elements in the Indy `NYM` object based on a series of steps. When a [[ref: NYM]] is created or updated the ledger MUST assemble the DIDDoc (following the steps) and validate the DIDDoc. As well, an Indy DID resolver will receive from the ledger the [[ref: NYM]] from the ledger and the non-validation steps must be followed to assemble the DIDDoc for the resolved DID.

Since the assembly validation may change between writing the DID and resolving it (notably, step 3.2), a client resolving a DID need SHOULD NOT perform the validation steps.

#### DIDDoc Assembly Steps

The following are the steps for assembling a DIDDoc from its inputs.

1. If the `verkey` is `null` the DID has been deactivated, and no DIDDoc is created. Assembly is complete; return a success status.
    1. *Note: On creation, the operation would have failed on the "self-certifying DID" validation and not reached this point in the process.*
2. The Indy network instance `namespace`, the [[ref: NYM]] `dest` and the [[ref: NYM]] `verkey` items are merged into a text template to produce a base DIDDoc.
    1. See the template in the [Base DIDDoc Template](#base-diddoc-template) section of this document.
    2. If there is no `diddocContent` item in the [[ref: NYM]], assembly is complete; return the DIDDoc and a success status.
3. If the `diddocContent` item is included in the [[ref: NYM]] is verified and merged into the DIDDoc.
    1. The `diddocContent` item MUST NOT have an `id` item. If found, exit the assembly process, returning an error.
    2. If the `diddocContent` item contains a `@context` item, the DIDDoc is assumed to be JSON-LD, and the `@context` element MUST include the current DID Core JSON-LD context. If it does not, exit the assmebly process and return an error.
    3. If the `diddocContent` item contains `verificationMethod` and/or `authentication` items, process them as follows.
        1. The entries MUST NOT have the same `id` values as those from the [[ref: NYM]]-generated DIDDoc. If a matching `id` is found, exit and return an error.
        2. Merge the entries into the respective arrays of DIDDoc.
    4. Add the other items of the `diddocContent` to the DIDDoc.
4. The resulting DIDDoc text must be valid JSON. If not JSON, exit and return an error.
5. The resulting JSON must be a valid DIDDoc. Perform a `<to be determined>` validation process. If not a DIDDoc, exit and return an error.
::: warning is this the correct process name
is `<to be determined>` the correct process name?
:::
6. Return the DIDDoc and a success status.

The remainder of this section goes through examples of base DIDDoc template (step 2, above) that is created prior to processing the optional `diddocContent` item, and an example of processing a `diddocContent` item.

##### Base DIDDoc Template

The base DIDDoc template is static text that forms a JSON structure. To transform a [[ref: NYM]] to its minimal DIDDoc, the Indy network instance's `namespace`, and the [[ref: NYM]] values `dest` and `verkey` are inserted into the template as indicated below.

```json
{
  "id": "did:indy:<namespace>:<dest>",
  "verificationMethod": [{
      "id": "did:indy:<namespace>:<dest>#verkey",
      "type": "Ed25519VerificationKey2018",
      "publicKeyBase58": "<verkey>",
      "controller": "did:indy:<namespace>:<dest>"
    }
  ],
  "authentication": [
    "did:indy:<namespace>:<dest>#verkey"
  ]
}
```

Assuming values `sovrin` for the `namespace`, `123456` for `dest` and `789abc` for the `verkey` the resulting JSON DIDDoc would be:

::: example Base DIDDoc sovrin example
```json
{
  "id": "did:indy:sovrin:123456",
  "verificationMethod": [{
      "id": "did:indy:sovrin:123456#verkey",
      "type": "Ed25519VerificationKey2018",
      "publicKeyBase58": "789abc",
      "controller": "did:indy:sovrin:123456"
    }
  ],
  "authentication": [
    "did:indy:sovrin:123456#verkey"
  ]
}
```
:::

##### Example Extended DIDDoc Item

An example of a [[ref: NYM]]'s extended DIDDoc handling is provided below. In the example below, the `diddocContent` item adds a DIDcomm Messaging service endpoint to the resolved DIDDoc. Note that in the example, an `@context` item is included in the `diddocContent`, which causes the result DIDDoc to be a JSON-LD document, rather than plain JSON.

::: example Extended DIDDoc Item example
```jsonld
"diddocContent" : {
    "@context" : [ 
        "https://www.w3.org/ns/did/v1",
        "https://identity.foundation/didcomm-messaging/service-endpoint/v1"
    ],
    "serviceEndpoint": [
        {
          "id": "did:indy:sovrin:123456#didcomm",
          "type": "didcomm-messaging",
          "serviceEndpoint": "https://example.com",
          "recipientKeys": [ "#verkey" ],
          "routingKeys": [ ]
        }
    ]
  }
```
:::

Applying the DIDDoc assembly rules to the example above produces the following assembled, valid JSON-LD DIDDoc:

::: example assembled Extended JSON-LD DIDDoc Item  example
```jsonld
{
  "@context": [
    "https://www.w3.org/ns/did/v1",
    "https://identity.foundation/didcomm-messaging/service-endpoint/v1"
  ],
  "id": "did:indy:sovrin:123456",
  "verificationMethod": [{
      "id": "did:indy:sovrin:123456#verkey",
      "type": "Ed25519VerificationKey2018",
      "publicKeyBase58": "789abc",
      "controller": "did:indy:sovrin:123456"
    }
  ],
  "authentication": [
    "did:indy:sovrin:123456#verkey"
  ],
  "serviceEndpoint": [
    {
      "id": "did:indy:sovrin:123456#didcomm",
      "type": "didcomm-messaging",
      "serviceEndpoint": "https://example.com",
      "recipientKeys": [ "#verkey" ],
      "routingKeys": []
    }
  ]
}
```
:::

#### Key Agreement

By default there is no key agreement section in an assembled DIDDoc. If the DID Controller wants a key agreement key in the DIDDoc, they must explicitly add it by including it in the `diddocContent`. For an ED25519 verification key, an X25519 key agreement key could be derived from the verkey (by the client), or a new key agreement key can be generated and used.

#### The "endpoint" ATTRIB

Prior to the definition of this DID Method, a convention on Indy ledgers to associate an endpoint to a [[ref: NYM]] involved adding an [[ref: ATTRIB]] ledger object with a `raw` value of contain the JSON for a name-value pair of `endpoint` and a URL endpoint, often an IP address.

We recommend that anyone that is using that format prior to the availability of the `did:indy` DID Method update their [[ref: NYM]] on the ledger to use the `diddocContent` item as soon as possible.

If a client retrieves a [[ref: NYM]] that has a `diddocContent` data element, the client should assume that the DID Controller has made the [[ref: ATTRIB]] (if any) obsolete and the client SHOULD NOT retrieve the [[ref: ATTRIB]] associated with the DID.

If clients want to continue to retrieve and use the `endpoint` [[ref: ATTRIB]] transaction associated with a [[ref: NYM]], we recommend that the endpoint value (along with `namespace` and `dest`) be used as if the following was the `diddocContent` item in the [[ref: NYM]].

``` json
"diddocContent" : {
    "@context" : [ "https://identity.foundation/didcomm-messaging/service-endpoint/v1" ],
    "serviceEndpoint": [
        {
          "id": "did:indy:<namespace>:<dest>#didcomm",
          "type": "didcomm-messaging",
          "serviceEndpoint": "<ENDPOINT>",
          "recipientKeys": [ "#verkey" ]
          "routingKeys": [ ]
        }
    ]
  }
```

The DIDDoc produced by the [[ref: NYM]] and "endpoint" [[ref: ATTRIB]] would be created using the DIDDoc Assembly Rules and using the `diddocContent` from the [[ref: ATTRIB]] instead of the [[ref: NYM]] item.

### Update

Updating a DID using the Indy DID Method occurs when a `NYM` transaction is performed by the [[ref: NYM]]'s controller (the "owner" of the [[ref: NYM]]) using the same identifier (`dest`). The Indy ledger MUST validate the [[ref: NYM]] transaction prior to writing the [[ref: NYM]] to the ledger.

When a [[ref: NYM]] is updated, the identifier (`dest`) for the [[ref: NYM]] does not change, but other values, including the `verkey` and `diddocContent`, may be changed. This means that (as expected) the DID itself does not change, but the DIDDoc returned by the DID may change.

The following validation steps are performed prior to the update being written to the ledger:

- Based on the configured authorization rules of the specific Indy ledger, the transaction may have to be signed by others, such as a Trustee or Endorser. If transaction is not authorized, the transaction MUST be rejected and an error returned to the client.

- The [[ref: NYM]] transaction requires that the transaction to be written is signed by the DID controller using the existing `verkey` and, if a new `verkey` is being provided, the new `verkey`. The ledger MUST verify the signature(s). If the signature(s) cannot be validated, the transaction MUST be rejected and an error returned to the client.

- The Indy ledger MUST check that the data in the [[ref: NYM]] produces valid JSON and MUST do a limited DIDDoc validation check prior to writing the [[ref: NYM]] object to the ledger. Details of the assembly and verification are [below](#DIDDoc-Assembly-and-Verification). If the DIDDoc validation fails, the transaction MUST be rejected and an error returned to the client.

Although the DIDDoc is returned from the DIDDoc assembly and verification process, the DIDDoc is not used further by the ledger.

Once the validation checks are completed, the [[ref: NYM]] update transaction is written to the Indy distributed ledger. If the [[ref: NYM]] write operation fails, an error MUST be returned to the client.

On successfully writing the update transaction to the Indy distributed ledger a success status is returned to the client.

### Read

Reading (resolving) a `did:indy` DID requires finding and connecting to the Indy ledger instance holding the DID, retrieving the [[ref: NYM]] associated with the DID, verifying the state proof for the returned [[ref: NYM]], and then assembling the DIDDoc. A client/resolver must perform the following steps to complete the process.

1. Given a `did:indy` DID, extract the `<namespace>` component of the DID.
2. If the namespace (specific Indy instance) is known to the resolver and the resolver is connected to the ledger continue. If not:
    1. To read the DIDDoc, the client must get the genesis file for the Indy instance and connect to the ledger. For example, the guidance in the [finding Indy ledgers](9_finding_indy_ledgers.md#Finding-Indy-Ledgers) can be used to discover previously unknown Indy ledgers.
    2. If the client cannot find the Indy network terminate the process and return a "Not Found" status to the caller.
    3. If the client chooses not to connect to the Indy network terminate the process and return a "Not Authorized" status to the caller.
3. Once connected, the `GET_NYM` Indy request is used, passing in the `<namespace identifer>` component.
    1. If resolving a prior version of the DID, a different call is used in at this point. See the [DID Versions](#DID-Versions) section of this document (below) for more details.
4. If the call fails, terminate the process and return a "Not Found" status to the caller.
5. If a [[ref: NYM]] is returned, use the state proof to verify the result. If the verification fails, terminate the process and return a "Not Found" status to the caller.
6. Use the DID `<namespace>` component, the [[ref: NYM]] data items `dest`, `verkey`, and (optional) `diddocContent` to assemble the DIDDoc using the [DIDDoc assembly process](#DIDDoc-Assembly-and-Verification) defined earlier in this document.
    1. Since the assembly validation was done by the ledger before writing the document, the process should be successful.
7. If the DIDDoc is empty (because the `verkey` is null) return a "Not Found" result, otherwise, return the DIDDoc.

#### DID Versions

In resolving a `did:indy` DID, the DID resolution query parameters `versionId` and `versionTime` may be used. When used, process to retrieve the [[ref: NYM]] from the ledger (step 3 above) is different.

If the parameter `versionId` is used, the value must be an Indy ledger `seqno` for the requested [[ref: NYM]] on the queried Indy ledger. Instead of using the `GET_NYM` call, the Indy `GET_TXN` call is used, passing in the `seqno`. The result is checked that it is the [[ref: NYM]] matching the DID namespace identifier. If so the call is considered to have failed. Either way, the process continues at Step 4.

If the parameter `versionTime` is used, the `GET_NYM` transaction is called with the `versionTime` timestamp as an additional parameter. The Indy ledger code tries to find the instance of the requested [[ref: NYM]] that was active at that time (using ledger transaction timestamps) and returns it (if found) or the call fails. Either way, the process continues at Step 4.

### Deactivate

Deactivtion of a `did:indy` DID is done by setting the [[ref: NYM]] verkey to null. Once done, the DIDDoc is not found (per 
the [DIDDoc Assembly Rules](#DIDDoc-Assembly-Steps)) and the [[ref: NYM]] cannot be updated again.

::: warning dead link
section `#diddoc-assembly-rules` does not exist. Pointing to `#DIDDoc-Assembly-Steps`
:::