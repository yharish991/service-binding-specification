## Binding Metadata in Annotations

 During a binding operation, annotations from relevant Kubernetes resources are extracted to gather information about what is interesting for binding. This information is eventually used to bind the application with the backing service by populating the binding Secret.

### Requirements for specifying binding information in a backing service CRD / Kubernetes resource

1. Extract a string from the Kubernetes resource.
2. Extract a string from the Kubernetes resource, and map it to custom name in the binding Secret.
3. Extract an entire configmap/Secret from the Kubernetes resource.
4. Extract a specific field from the configmap/Secret from the Kubernetes resource, and bind it as an environment variable.
5. Extract a specific field from the configmap/Secret from the Kubernetes resource and and bind it as a volume mount.
6. Extract a specific field from the configmap/Secret from the Kubernetes resource and map it to different name in the binding Secret.
7. Extract a “slice of maps” from the Kubernetes resource and generate multiple fields in the binding Secret.
8. Extract a "slice of strings" from a Kubernetes resource and indicate the content in a specific index in the slice to be relevant for binding.


### Data model : Building blocks for expressing binding information

* `path`: A template representation of the path to the element in the Kubernetes resource. The value of `path` could be specified in either [JSONPath](https://kubernetes.io/docs/reference/kubectl/jsonpath/) or [GO templates](https://golang.org/pkg/text/template/)

* `elementType`: Specifies if the value of the element referenced in `path` is of type `string` / `sliceOfStrings` / `sliceOfMaps`. Defaults to `string` if omitted.

* `objectType`: Specifies if the value of the element indicated in `path` refers to a `ConfigMap`, `Secret` or a plain string in the current namespace!  Defaults to `Secret` if omitted and `elementType` is a non-`string`.

* `bindAs`: Specifies if the element is to be bound as an environment variable or a volume mount using the keywords `envVar` and `volume`, respectively. Defaults to `envVar` if omitted.

* `sourceKey`: Specifies the key in the configmap/Secret that is be added to the binding Secret. When used in conjunction with `elementType`=`sliceOfMaps`, `sourceKey` specifies the key in the slice of maps whose value would be used as a key in the binding Secret. This optional field is the operator author intends to express that only when a specific field in the referenced `Secret`/`ConfigMap` is bindable.

* `sourceValue`: Specifies the key in the slice of maps whose value would be used as the value, corresponding to the value of the `sourceKey` which is added as the key, in the binding Secret. Mandatory only if `elementType` is `sliceOfMaps`.


### A Sample CR : The Kubernetes resource that the application would bind to

```
    apiVersion: apps.kube.io/v1beta1
    kind: Database
    metadata:
    name: my-cluster
    spec:
    ...
    status:
        bootstrap:   
            - type: plain   
    	      url: myhost2.example.com
              name: hostGroup1
            - type: tls
    	      url: myhost1.example.com:9092,myhost2.example.com:9092
              name: hostGroup2
        data:
            dbConfiguration: database-config  # configmap 
            dbCredentials: database-cred-Secret # Secret
            url: db.stage.ibm.com
```



### Scenarios


1. #### Use everything from the Secret  “status.data.dbCredentials” 

    Requirement : *Extract an entire configmap/Secret from the Kubernetes resource*


    Annotation:

    ```
    “servicebinding.dev/dbcredentials”:”path={.status.data.dbcredentials},objectType=Secret”
    ```


    Descriptor:

    ```
    - path: data.dbcredentials
      x-descriptors:
        - urn:alm:descriptor:io.kubernetes:Secret 
        - servicebinding
    ```


2. #### Use everything from the ConfigMap “status.data.dbConfiguration” 


    Requirement : *Extract an entire configmap/Secret from the Kubernetes resource*

    Annotation

    ```
    “servicebinding.dev/dbConfiguration”: "path={.status.data.dbConfiguration},objectType=ConfigMap”
    ```


    Descriptor

    ```
    - path: data.dbConfiguration
      x-descriptors:
        - urn:alm:descriptor:io.kubernetes:ConfigMap 
        - servicebinding
    ```

3. #### Use “certificate” from the ConfigMap “status.data.dbConfiguration” as an environment variable

    Requirement : *Extract a specific field from the configmap/Secret from the Kubernetes resource and use it as an environment variable.*


    Annotation

    ```
    “servicebinding.dev/certificate”:
    "path={.status.data.dbConfiguration},objectType=ConfigMap"
    ```


    Descriptor


    ```
    - path: data.dbConfiguration
      x-descriptors:
        - urn:alm:descriptor:io.kubernetes:ConfigMap 
        - servicebinding:certificate:bindAs=envVar
    ```


4. #### Use “certificate” from the ConfigMap “status.data.dbConfiguration” as a volume mount

    Requirement : *Extract a specific field from the configmap/Secret from the Kubernetes resource and use it as a volume mount.*


    Annotation

    ```
    “servicebinding.dev/certificate”:
    "path={.status.data.dbConfiguration},bindAs=volume,objectType=ConfigMap"
    ```


    Descriptor      

    ```
    - path: data.dbConfiguration
      x-descriptors:
        - urn:alm:descriptor:io.kubernetes:ConfigMap 
        - servicebinding:certificate:bindAs=volume
    ```


5. #### Use “db_timeout” from the ConfigMap “status.data.dbConfiguration” as “timeout” in the binding Secret.

    Requirement: *Extract a specific field from the configmap/Secret from the Kubernetes resource and map it to different name in the binding Secret*

    Annotation

    ```
    “servicebinding.dev/timeout”: 
    “path={.status.data.dbConfiguration},objectType=ConfigMap,sourceKey=db_timeout”
    ```


    Descriptor

    ```
    - path: data.dbConfiguration
      x-descriptors:
        - urn:alm:descriptor:io.kubernetes:ConfigMap 
        - servicebinding:timeout:sourceKey=db_timeout
    ```

6. #### Use the attribute “status.data.url”

    Requirement: *Extract a string from the Kubernetes resource.*

    Annotation

    ```
    “servicebinding.dev/uri”:"path={.status.data.url}"
    ```

    Descriptor

    ```
    - path: data.uri
      x-descriptors:
        - servicebinding
    ```

7. #### Use the attribute “status.data.“connectionURL” as uri in the binding Secret

    Requirement: *Extract a string from the Kubernetes resource, and map it to custom name in the binding Secret.*

    Annotation

    ```
    “servicebinding.dev/uri: "path={.status.data.connectionURL}”
    ```



    Descriptor

    ```
    - path: data.connectionURL
      x-descriptors:
        - servicebinding:uri
    ```

8. #### Use specific elements from the CR’s “status.bootstrap” to produce key/value pairs in the  binding Secret

    Requirement: *Extract a “slice of maps” from the Kubernetes resource and generate multiple fields in the binding Secret.*

    Annotation

    ```
    “servicebinding.dev/endpoints”:
    "path={.status.bootstrap},elementType=sliceOfMaps,sourceKey=type,sourceValue=url" 
    ```


    Descriptor

    ```
    - path: bootstrap
      x-descriptors:
        - servicebinding:endpoints:elementType=sliceOfMaps:sourceKey=type:sourceValue=url
    ```