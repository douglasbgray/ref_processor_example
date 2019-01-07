# ref_processor_example

This sample demonstrates an issue with the reference processor in the Swagger parser.

When there is a map property with a map property, and the final type is a reference in another file, then the
 reference processor does not normalize the reference path relative to the file location of the
 root contract. This causes a file not found error when running the code generator. The reference should
 have been updated from `./types.yaml` to `./types/types.yaml`, which did not happen.

```
Caused by: java.lang.RuntimeException: Could not find ./types.yaml on the classpath
    at io.swagger.v3.parser.util.ClasspathHelper.loadFileFromClasspath (ClasspathHelper.java:31)
    at io.swagger.v3.parser.util.RefUtils.readExternalRef (RefUtils.java:177)
    at io.swagger.v3.parser.ResolverCache.loadRef (ResolverCache.java:118)
    at io.swagger.v3.parser.processors.ExternalRefProcessor.processRefToExternalSchema (ExternalRefProcessor.java:51)
    at io.swagger.v3.parser.processors.SchemaProcessor.processReferenceSchema (SchemaProcessor.java:202)
    at io.swagger.v3.parser.processors.SchemaProcessor.processAdditionalProperties (SchemaProcessor.java:74)
    at io.swagger.v3.parser.processors.SchemaProcessor.processSchemaType (SchemaProcessor.java:61)
    at io.swagger.v3.parser.processors.SchemaProcessor.processArraySchema (SchemaProcessor.java:188)
    at io.swagger.v3.parser.processors.SchemaProcessor.processSchemaType (SchemaProcessor.java:47)
    at io.swagger.v3.parser.processors.SchemaProcessor.processPropertySchema (SchemaProcessor.java:105)
    at io.swagger.v3.parser.processors.SchemaProcessor.processSchemaType (SchemaProcessor.java:54)
    at io.swagger.v3.parser.processors.SchemaProcessor.processSchema (SchemaProcessor.java:39)
    at io.swagger.v3.parser.processors.ComponentsProcessor.processSchemas (ComponentsProcessor.java:222)
    at io.swagger.v3.parser.processors.ComponentsProcessor.processComponents (ComponentsProcessor.java:72)
    at io.swagger.v3.parser.OpenAPIResolver.resolve (OpenAPIResolver.java:50)
    at io.swagger.v3.parser.OpenAPIV3Parser.readLocation (OpenAPIV3Parser.java:53)
    at io.swagger.parser.OpenAPIParser.readLocation (OpenAPIParser.java:19)
```

See the [test contract](src/main/resources/types/responses.yaml) which has different combinations
 of array and map schemas. To reproduce, clone this project and execute.

```
# Code generation requires maven 3.2.5 or later.
mvn clean compile
```

In researching, the problem lies in an [incomplete implementation](https://github.com/swagger-api/swagger-parser/blob/master/modules/swagger-parser-v3/src/main/java/io/swagger/v3/parser/processors/ExternalRefProcessor.java#L180-L212) of the method `processProperties` in
 the `ExternalRefProcessor`. The method does not recurse in all situations, so depending on the contract, it can lead to
 an incomplete reference resolution.

I was able to fix the issue by replacing that method with these 3 methods:

```
    private void processProperty(Schema subProp, String file) {
        if (subProp != null) {
            if (StringUtils.isNotBlank(subProp.get$ref())) {
                processRefSchema(subProp, file);
            }
            if (subProp.getProperties() != null) {
                processProperties(subProp.getProperties(), file);
            }
            if (subProp instanceof ArraySchema) {
                processProperty(((ArraySchema) subProp).getItems(), file);
            }
            if (subProp.getAdditionalProperties() instanceof Schema) {
                processProperty(((Schema) subProp.getAdditionalProperties()), file);
            }
            if (subProp instanceof ComposedSchema) {
                ComposedSchema composed = (ComposedSchema) subProp;
                processProperties(composed.getAllOf(), file);
                processProperties(composed.getAnyOf(), file);
                processProperties(composed.getOneOf(), file);
            }
        }
    }

    private void processProperties(Collection<Schema> subProps, String file) {
        if (subProps != null) {
            for (Schema subProp : subProps) {
                processProperty(subProp, file);
            }
        }
    }

    private void processProperties(Map<String, Schema> subProps, String file) {
        if (subProps != null) {
            processProperties(subProps.values(), file);
        }
    }
```
