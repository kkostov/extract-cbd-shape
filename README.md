# Extract CBD Shape

Given (i) an N3 Store of triples, (ii) an N3 Store with a SHACL shape’s triples, and (iii) a target entity URI,
this library will extract all triples that belong to the entity.
If more triples of the entity are needed, extra triples are retrieved by dereferencing the relevant entity.

The algorithm is a proposal to be standardized as part of [W3C’s TREE hypermedia Community Group](https://w3id.org/tree/specification) as the member extraction algorithm. This algorithm needs to be efficient and unambiguously defined, so that various implementations of the member extraction algorithm will result in the same set of triples. As a trade-off, the resulting set of triples is not guaranteed to be validated by the SHACL shape.

The algorithm is inspired by, and an in-between between [CBD](https://www.w3.org/Submission/CBD/) and [Shape Fragments](https://github.com/Shape-Fragments/old-shapefragments-paper/blob/main/fullpaper.pdf), thanks to [Thomas Bergwinkl and his blog post](https://www.bergnet.org/2023/03/2023/shacl-engine/) on a SHACL engine.

## Use it

```bash
## Package not yet available on NPM
npm install extract-cbd-shape
```

```javascript
import {CBDShapeExtractor} from "extract-cbd-shape";
// ...
let extractor = new CBDShapeExtractor(shapesGraph);
let entityquads = await extractor.extract(store, entityId, shapeId);
```

## Test it

Tests and examples provided in the [tests](tests/) library. Run them using mocha which can be invoked using `npm test`

## Algorithm and limitations

This is an extension of CBD. It extracts:
 1. all quads with subject this entity, and their blank node triples (recursively)
 2. all quads with a named graph matching the entity we’re looking up

To be discussed:
 3. _Should it also extract all RDF reification quads?_ (Included in the original CBD)
 4. _Should it also extract all singleton properties?_
 5. _Should it also extract RDF* annnotations?_

If no triples were found based on CBD, it does an HTTP request to the entity’s IRI (fallback to IRI dereferencing)

Next, it takes _hints_ (it does not guarantee a result that validates) from an optional SHACL shapes graph. It only uses the parts relevant for discovery for the [SHACL Core Constraint Components](https://www.w3.org/TR/shacl/#core-components). It does not support SPARQL or Javascript.
 1. Checks if the Shape is deactivated first
 2. All links to nodes that are given, in conditionals or not, are added to the shape’s NodeLinks array. The paths matched by the nodelinks will be processed on that matching namednode in the data
 3. Processes all `sh:property` links to property shapes. Only marks a property as required if `sh:minCount` > 0. It does not validate cardinalities.
 4. Processes the conditionals `sh:xone`, `sh:or` and `sh:and` (but doesn’t process `sh:not` -- See https://www.w3.org/TR/shacl/#core-components-logical):
     * `sh:and` doesn’t have much effect: all nodelinks and all required paths will be processed
     * `sh:xone` and `sh:or`: in both cases, at least one item must match at least one quad for all required paths. If not, it will do an HTTP request to the current namednode. It’s thus becomes less likely an HTTP will be performed with an `sh:xone` or `sh:or` involved: if one item’s condition of required paths has been fulfilled, it doesn’t search further. This is a design trade-off: we don’t allow more complex SHACL constraints (more complex than checking the required paths) to trigger HTTP requests. 
 4. It processes full `sh:path` and __includes the quads necessary to reach their targets__.

It won’t:
 1. Process more complex validation instructions that are part of SHACL such as `sh:class`, inLanguage, pattern, value, qualified value shapes, etc. It is the data publisher’s responsibility to provide valid data, or it is the responsibility of the user of the library to validate the quads afterwards.
 2. Do automatic target selection based on e.g., targetClass: you need to set the target.
